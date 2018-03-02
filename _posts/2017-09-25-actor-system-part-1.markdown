---
layout: post
title:  "Actor System with Akka Part 1"
date:   2017-09-25 18:42:21
categories: jekyll update
key: actor_system_example
comment: true
---
## The Problem

At [Lastmilelink](https://lastmilelink.com) we need to manage a large fleet of couriers to get goods delivered to customers efficiently.  On a typical day there are thousands of deliveries, with customers, controllers and couriers constantly sending data to the backend system.

Some of the biggest challenges of processing the data related to couriers are:

1. How do we know which couriers are available?
2. Given a location, which couriers are closest to it?

These problems may seem easy, but are actually not when coupled with the requirements:

#### Resilience to failures

No system runs forever.  A server failing shouldn't take down the whole functionality.

#### Horizontal scaling

The system needs to scale out into multiple servers, with load distributed among them.

#### Data consistency

The system should look consistent from the outside.  It shouldn't give different information about the same thing (e.g., a courier's location).

#### Persistence of state

If a disaster happens and the whole cluster is down, the state can be used to restore a new system.

This is also useful when doing a downtime upgrade.

## Demo: Courier Recommendation Prototype

[![undefined]({{ site.url }}/assets/courier-realtime/youtube.png)](https://www.youtube.com/watch?v=6HXvTO7jT9E)

Note how the prototype fulfills the above requirements:

* There is no single point of failure (system states are migrated if a node dies)
* New instances can be added and system states are automatically balanced
* Data is never lost
* Data is always consistent, e.g., the system never reports a courier as in two different cells

### Components

![]({{ site.url }}/assets/courier-realtime/courier-realtime-architecture.png)

### Load Test

Before we dive into details, let's see how the system performs under heavy load:

* 1000 x 1000 grid
* 10000 restaurants
* 2000 couriers (1 ping per minute)
* Service: 2 x c4.large EC2 instances (2 vCPU, 3.75GB RAM each)
* Service JVM: 2GB heap, CMS GC
* Cassandra: 3 x c4.large, replication factor = 3
* Cassandra JVM: 1GB heap, CMS GC

(All time units in milliseconds, yellow = 99.9th, purple = 99th, blue = 95th)
#### Courier Actors
![]({{ site.url }}/assets/courier-realtime/2000courier-courier-metrics.png)
#### Grid Actors
![]({{ site.url }}/assets/courier-realtime/2000courier-grid-metrics.png)
#### System
![]({{ site.url }}/assets/courier-realtime/2000courier-system-metrics.png)
#### Network
![]({{ site.url }}/assets/courier-realtime/2000courier-network-metrics.png)

#### Observations
* All messages are processed within ~10 milliseconds
* System load and network traffic are stable (the choppiness in UDP is due to some batching behavior)
* Read delay from Kinesis is stable and low (system copes with message rate)

## Introducing Actor Systems

![]({{ site.url }}/assets/courier-realtime/actors-illustration.png)
(http://www.brianstorti.com/the-actor-model/)

Definition of an actor:
* An entity that processes all messages sequentially
* Has internal states
* Communicate with each other by passing messages (never direct function calls)

And as a result:
* Actors are thread safe (no locking required in actor systems)
* Actors are cheap to create (can have millions per server node)
* Scheduling of actor execution is taken care by most libraries (so developers only need to focus on actor's internal logic and communication)

# [Akka](https://akka.io/) Architecture

Akka is an actor system library written in Scala.

![]({{ site.url }}/assets/courier-realtime/courier-realtime-akka-tech-stack.png)

Akka builds a wide range of features on top of actors, most of them were used in the prototype:

* Cluster: used to create a distributed system
* Remote: data transmission between instances
* Persistence: state persistence for actor migration and restart
* HTTP/WS: for communication with front end
* Pub/Sub: for sending/receiving system wide messages

## Code Example

Here is code of a very basic actor.  Note how messages are passed with the `!` syntax.

{% highlight scala %}
import akka.actor.Actor
import akka.actor.ActorSystem
import akka.actor.Props

class HelloActor extends Actor {
    def receive = {
        case "hello" => println("hello back at you")
        case _       => println("huh?")
    }
}

object Main extends App {
    val system = ActorSystem("HelloSystem")
    // default Actor constructor
    val helloActor = system.actorOf(Props[HelloActor], name = "helloactor")
    helloActor ! "hello"
    helloActor ! "buenos dias"
}
{% endhighlight %}

## Conclusion

This post shows a simple yet highly scalable model of monitoring a system with a large number of players, as well as introducing the actor system and Akka.

In the [next post]({% post_url 2017-09-30-actor-system-part-2 %}) we'll explain the details of the prototype, which will illustrate the philosophy of the actor system.
