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
 
* It does not involve knowledge of the queues content
* On the long term (not so long in human time), it allows to reach the message flow rate associated to a `Qe[i]`.

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


# RabbitMQ QoS

TBD

# Implementation


## Slot

Per queue we will define a consumer. RabbitMQ will push message to each consumer according to deque rate. 

First we define the slot. It contains the `quantum`, the `deficit` and the RabbitMQ blocking queue consumer. 

{% highlight java linenos %}
public class DwrrSlot {

    private final DwrrBlockingQueueConsumer dwrrBlockingQueueConsumer;
    private final int quantum;
    private int processedMsg;
    private int deficit;

    public DwrrSlot(DwrrBlockingQueueConsumer dwrrBlockingQueueConsumer, int quantum) {
        this.dwrrBlockingQueueConsumer = dwrrBlockingQueueConsumer;
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


## The slot consumer



{% highlight java linenos %}
public class DwrrBlockingQueueConsumer {

    private final BlockingDeque<QueueingConsumer.Delivery> deliveries;
    private final String queue;
    private final Channel channel;
    private final Semaphore availableToken;
    private final InternalConsumer consumer;


    public DwrrBlockingQueueConsumer(String queue, Channel channel, Semaphore availableToken, int prefetch) {
        this.queue = queue;
        this.channel = channel;
        this.availableToken = availableToken;
        this.deliveries = new LinkedBlockingDeque<>(prefetch);
        this.consumer = new InternalConsumer(channel);
    }

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
            deliveries.offer(new QueueingConsumer.Delivery(envelope, properties, body));
            availableToken.release();
        }
    }
}
{% endhighlight %}