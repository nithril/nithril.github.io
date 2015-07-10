---
layout: post
title:  "Fair Consuming With RabbitMQ"
date:   2015-07-05 10:18:46
categories: AMQP
comments: true
---

<img style="float: left;margin-right:20px;" src="/assets/2015-07-05-fair-consuming-with-rabbitmq/rabbitmq_logo.png">

This article will present a pattern to achieve fair consuming with RabbitMQ, 
an AMQP implementation using a deficit weighted round robin scheduler.
  
This article is not about RabbitMQ [Priority Queue Support](https://www.rabbitmq.com/priority.html).
A consumer bounds to a RabbitMQ priority queue will always consume first the messages with the highest priority.
It doesn't ensure that a message with a low priority will be processed, high priority messages may predate the processing slots.

   
<!--more-->


# ToC
* TOC
{:toc}


# The Pattern

We will implement the [Deficit Weighted Round Robin](https://en.wikipedia.org/wiki/Deficit_round_robin) algorithm (DWRR).
This algorithm is simple and effective:
 
* It does not involve knowledge of the actual queues content
* On the long term (not so long in human time), it allows to reach the targeted flow rate.

It involves knowledge of the past content: from which the word `Deficit`, past contents increase the deficit and thus decrease the scheduling priority.


DWRR is a scheduling algorithm for the network scheduler: the unit of resource is the network bandwidth. Packets are prioritized according to their size and the incoming/outgoing flow slot.
In our case, the unit of resource will be the processing time. The higher the processing time ratio is, the higher the message rate would be. This value is the `quantum`,
 it defines how much processing time ratio we will allocate per queue/slot. A quantum is noted `Q`.
  
The scheduling is done on queues. Given a priority range of `[1..N]`, it will involve `Qe[i]` queues. We will assign a quantum `Q[i]` to a queue `Qe[i]`. 
Quantum may be normalized to a ratio `Ratio[i] = Q[i] / Sum(Q[1..N])`. This ratio is the part of resource allocated for a queue. 


Pasts messages increase the deficit. Thus mesages must be weighted: a processing time cost must be computed per message. 
It can be constant or dynamic and it's depends on the resource type:

* Constant if the message processing time is constant
* Dynamic if per message it vary significantly 

For this article I will use a fixed weight.


**Example**

We have two queues: Q1, Q2. 
Q1 has a quantum of 10 and Q2 a quantum of 1. Q1 should have a processing rate 10 times faster than Q2. 
Taken a weight of 1, when Q1 deque 1 message per iteration, Q2 should wait 10 iteration to deque one.


# Implementation

The code is available on [github](https://github.com/nithril/article-fair-consuming-with-rabbitmq).

## Slot

Per queue we will define a consumer slot. RabbitMQ will push messages to each consumer according to deque rate.  
First we define the slot. It contains the `quantum`, the `deficit` and the RabbitMQ queue consumer. 

{% highlight java linenos %}
public class DwrrSlot {

    private final int quantum;
    private int deficit;
    private final DwrrBlockingQueueConsumer consumer;

    public DwrrSlot(DwrrBlockingQueueConsumer consumer, int quantum) {
        this.consumer = consumer;
        this.quantum = quantum;
    }

    public void reset(){
        deficit = 0;
    }

    public void reduceDeficit(){
        deficit = deficit + quantum;
    }

    public void increaseDeficit(int cost){
        deficit = deficit - cost;
    }
}
{% endhighlight %}


## The consumer

This class subscribes to a RabbitMQ queue and stores delivered messages into a collection (`deliveries`).
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
        //Subscribe to the queue and enable the acknowledgement
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

The `consumerToken#tryAcquire` allows to park the main loop thread if there is no available message.
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
                //Finally ack the message, so RabbitMQ will push a new one to the consumer
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


-----------------------------------

That's it. Pretty straightforward.
 

# RabbitMQ Consumer Prefetch

The AMQP supports an acknowledgement feature. When activated, the consumer must acknowledge (to the broker) all messages it receives.
 Until the message is unacked, RabbitMQ will not push a new message to the consumer. The consumer prefetch (aka QoS) configures how much unacked messages a consumer can hold. 

[Consumer prefetch](https://www.rabbitmq.com/consumer-prefetch.html) is an important RabbitMQ concept. 

> AMQP specifies the basic.qos method to allow you to limit the number of unacknowledged messages on a channel (or connection) when consuming (aka "prefetch count").

RabbitMQ will push message to the consumer until the number of unacked messages is reached. 


The prefetch value must be set according to the processing speed. The priority ratio will be flattened if the processing rate is greater than the RabbitMQ push rate because consumers will wait for messages most of the time 

> The goal is to keep the consumers saturated with work, but to minimise the client's buffer size so that more messages stay in Rabbit's queue and are thus available for new consumers or to just be sent out to consumers as they become free.

See this in depth article [*Some queuing theory: throughput, latency and bandwidth*](https://www.rabbitmq.com/blog/2012/05/11/some-queuing-theory-throughput-latency-and-bandwidth/)


# Results

**10 queues, a message weight of 4, each queue contains 40000 messages. The message processing time is set to 100µs**

`p9` deque rate is ten time faster than `p0` one. This ratio is equals to the quantum one.  

| Queue  | Qantum  | Consumed  | Rate      |
|:------:|---------|-----------|-----------|
| p0     | 4       | 3741      | 136.29408 |
| p1     | 8       | 7377      | 268.76276 |
| p2     | 12      | 11031     | 401.8872  |
| p3     | 16      | 14556     | 530.3119  |
| p4     | 20      | 18195     | 662.88983 |
| p5     | 24      | 21828     | 795.2492  |
| p6     | 28      | 25408     | 925.6777  |
| p7     | 32      | 29024     | 1057.4176 |
| p8     | 36      | 32605     | 1187.8826 |
| p9     | 40      | 36250     | 1320.6791 |



**10 queues, a message weight of 4, each queue contains 40000 messages. The message processing time is set to 1µs**

The rate is flattened. My RabbitMQ instance cannot sustain the processing time rate.  

| Queue  | Qantum  | Consumed  | Rate      |
|:------:|---------|-----------|-----------|
| p0     | 4       | 19546     | 1970.959  |
| p1     | 8       | 19323     | 1948.4723 |
| p2     | 12      | 20445     | 2061.6113 |
| p3     | 16      | 19750     | 1991.5297 |
| p4     | 20      | 20075     | 2024.3018 |
| p5     | 24      | 19780     | 1994.5548 |
| p6     | 28      | 20040     | 2020.7725 |
| p7     | 32      | 20227     | 2039.6289 |
| p8     | 36      | 20643     | 2081.5771 |
| p9     | 40      | 20188     | 2035.6963 |

