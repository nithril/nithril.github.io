---
layout: post
title:  "Fair Consuming With RabbitMQ"
date:   2015-06-28 10:18:46
categories: CR
comments: true
---

This article will present a pattern to achieve fair consuming with RabbitMQ, 
an AMQP implementation. It will involve weighted queues bound to a consumer which will use fair scheduling.
  
This article is not about RabbitMQ [Priority Queue Support](https://www.rabbitmq.com/priority.html).
A consumer bounds to a RabbitMQ priority queue will always consume first the messages with the highest priority.
It doesn't ensure that a message with a low priority will be processed. Indeed high priority message may predate the processing slots.

   
<!--more-->


# ToC
* TOC
{:toc}


# The Pattern

We will split the priority per queue. Given a priority range of `[1..N]`, it will involve `Qe[i]` queues.

We will implement the [Deficit Weighted Round Robin](https://en.wikipedia.org/wiki/Deficit_round_robin) algorithm (DWRR).
This algorithm is simple and effective:
 
* It does not involve knowledge of the actual queues content
* On the long term (not so long in human time), it allows to reach the message flow rate associated to a `Qe[i]`.

It involves knowledge of the past content. It is why it's called `Deficit`, past contents decrease the scheduling priority.


DWRR is a scheduling algorithm for the network scheduler: the unit of resource is the network bandwidth. Packets are prioritized according to their size and the flow slot.

In our case, the unit of resource is the processing time. The higher the processing time ratio is, the higher the message rate would be. This value is the `quantum`, 
it defines how much processing time ratio we will allocate per queue/slot. A quantum is noted `Q[i]` and is normalized to a ratio `Ratio[i] = Q[i] / Sum(Q[1..N])`.
  
Messages must be weighted: a processing time cost. It can be constant or dynamic and it's depends on the resource type. For example:

* Constant if the message processing time is constant
* Dynamic if per message it vary significantly 

For this article I will use a fixed weight.

Two queues: Q1, Q2. 
Q1 has a quantom of 10 and Q2 a quantum of 1. Q1 should have a processing rate 10 times faster than Q2. 
Taken a processing time cost of 1, when Q1 deque 1 message per iteration, Q2 should wait 10 iteration to deque one.


# Implementation


## Slot

Per queue we will define a consumer slot. RabbitMQ will push messages to each consumer according to deque rate.  
First we define the slot. It contains the `quantum`, the `deficit` and the RabbitMQ queue consumer. 

{% highlight java linenos %}
public class DwrrSlot {

    private final int quantum;
    private int deficit;
    private final DwrrBlockingQueueConsumer consumer;
    private int processedMsg;

    public DwrrSlot(DwrrBlockingQueueConsumer consumer, int quantum) {
        this.consumer = consumer;
        this.quantum = quantum;
    }

    public void reset(){
        deficit = 0;
    }

    public int reduceDeficit(){
        deficit = deficit + quantum;
        return deficit;
    }

    public int increaseDeficit(int cost){
        deficit = deficit - cost;
        processedMsg = processedMsg + 1;
        return deficit;
    }
}
{% endhighlight %}


## The consumer

This class subscribes to a RabbitMQ queue and stores delivered messages into `deliveries` collection.
The `consumerToken` semaphore releases a consumer token for each delivered message. Thus the main loop thread
can be notified when a new message is available.

{% highlight java linenos %}
public class DwrrBlockingQueueConsumer {

    private final BlockingDeque<QueueingConsumer.Delivery> deliveries;
    private final String queue;
    private final Channel channel;
    private final Semaphore consumerToken;
    private final InternalConsumer consumer;


    public DwrrBlockingQueueConsumer(String queue, Channel channel, Semaphore consumerToken, int prefetch) {
        this.queue = queue;
        this.channel = channel;
        this.consumerToken = consumerToken;
        this.deliveries = new LinkedBlockingDeque<>(prefetch);
        this.consumer = new InternalConsumer(channel);
    }

    /**
     * Start to consume messages from the queue
     * @throws IOException
     */
    public void start() throws IOException {
        channel.basicConsume(queue, false, consumer);
    }

    public BlockingDeque<QueueingConsumer.Delivery> getDeliveries() {
        return deliveries;
    }

    private class InternalConsumer extends DefaultConsumer {
        public InternalConsumer(Channel channel) {
            super(channel);
        }
        @Override
        public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) {
            //Queue the message
            deliveries.offer(new QueueingConsumer.Delivery(envelope, properties, body));
            //Release a consumer token
            consumerToken.release();
        }
    }
}
{% endhighlight %}


## The main loop

We see in the main loop the purpose of the `consumerToken` semaphore. The `tryAcquire` allows to park the main loop thread if there is no delivery.
The release of a `consumerToken` by `DwrrBlockingQueueConsumer.InternalConsumer#handleDelivery` will wake up the main loop thread with a minimal delay. 

{% highlight java linenos %}
while (true) {
    int processedMessageCounter = 0;

    //Try to acquire a consumer token, wait for 5 seconds
    if (consumerToken.tryAcquire(1, 5, TimeUnit.SECONDS)) {
        //We got a token, a delivery is available
        for (DwrrSlot slot : slots) {
            //The deficit is reduced per iteration
            slot.reduceDeficit();

            //Loop until the slot does not contain any deliveries or until the slot deficit is bellow the message weight
            while (!slot.getConsumer().getDeliveries().isEmpty() && slot.getDeficit() >= MESSAGE_WEIGHT) {
                QueueingConsumer.Delivery delivery = slot.getConsumer().getDeliveries().poll();
                slot.increaseDeficit(MESSAGE_WEIGHT);
                //Simulate processing time
                Thread.sleep(0, 1000);
                //Finally ack the message, so RabbitMQ will push a new one to the slot consumer
                channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
            }
            //If the slot does not contain any deliveries, reset the deficit
            if (slot.getConsumer().getDeliveries().isEmpty()) {
                slot.reset();
            }
        }
        
        if (processedMessageCounter > 0) {
            //If we have processed message, we must acquire the number of processed message minus the first acquire
            consumerToken.acquire(processedMessageCounter - 1);
        } else {
            //If we do not have processed message (because all deficit where < weight) we must release the token
            consumerToken.release();
        }

    } else {
        LOG.info("No message");
    }
}
{% endhighlight %}


# RabbitMQ Consumer Prefetch

[Consumer prefetch](https://www.rabbitmq.com/consumer-prefetch.html) is an important RabbitMQ concept
> AMQP specifies the basic.qos method to allow you to limit the number of unacknowledged messages on a channel (or connection) when consuming (aka "prefetch count").

RabbitMQ will push message to the consumer until the number of unacked messages is reached. 
The prefetch value must be set according to the processing speed. The priority ratio will be flattened if the processing rate is greater than the RabbitMQ push rate because consumers will wait for messages most of the time 

> The goal is to keep the consumers saturated with work, but to minimise the client's buffer size so that more messages stay in Rabbit's queue and are thus available for new consumers or to just be sent out to consumers as they become free.

See this in depth article [*Some queuing theory: throughput, latency and bandwidth*](https://www.rabbitmq.com/blog/2012/05/11/some-queuing-theory-throughput-latency-and-bandwidth/)


# Results


Name=p0 Quantum=4  Consumed msg=3741 Msg/s=136.29408
Name=p1 Quantum=8  Consumed msg=7377 Msg/s=268.76276
Name=p2 Quantum=12 Consumed msg=11031 Msg/s=401.8872
Name=p3 Quantum=16 Consumed msg=14556 Msg/s=530.3119
Name=p4 Quantum=20 Consumed msg=18195 Msg/s=662.88983
Name=p5 Quantum=24 Consumed msg=21828 Msg/s=795.2492
Name=p6 Quantum=28 Consumed msg=25408 Msg/s=925.6777
Name=p7 Quantum=32 Consumed msg=29024 Msg/s=1057.4176
Name=p8 Quantum=36 Consumed msg=32605 Msg/s=1187.8826
Name=p9 Quantum=40 Consumed msg=36250 Msg/s=1320.6791







