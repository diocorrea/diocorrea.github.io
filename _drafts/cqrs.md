---
layout: post
title: Command Query Responsibility Segregation [CQRS]
categories: [patterns,DDD]
tags: [CQRS,command-query-responsibility-segregation,scaling, domain-driven-design,DDD]
mermaid: true
---
There are moments where the patterns used for querying the models of an application are far different from those used for managing its life cycle. The `CQRS` pattern talks about that, how to separate your changes to the model from different query access. We will go over an example on this post on when this happen and a couple of alternatives to how to improve your system scale on such situation.

## Background

You manage a system that is responsible for managing blog posts, a Content Management System, CMS. This system has two main functionalities, it allows writers to create and manage content, CRUD, and also allows readers to consume  the final content. You can assume that once the `Article` is published it will be consumed by a high amount of readers. 

## Problem

Starting from using a simple model, where a Sql database is used and the same tables are shared between the writer and reader use cases, over time becomes a bottleneck. One functionality impacts the other in terms of performance and implementing new features becomes harder due to the many different scenarios that needs to be taken into consideration.

![overview](/assets/img/diagrams/cqrs/simple-deployment.excalidraw.png)

## Identifying bounded contexts

The first thing you should do is identify the different contexts that we we have represented in our application. In this particular example maybe could be easy to spot them, let's assume some use cases like:

* Reader:
  * might want to use textual search to find the posts
  * might want have them pre-rendered on the backend stead of sending Json to the front end to improve customer experience
  * reference to custom tags
* Writer:
  * might want to be able to store draft version of the posts
  * might want to schedule a publish time

> There are scientific ways of find these bounded contexts, you could use for instance an [`Event Storming`](https://www.eventstorming.com/) session. This is not the focus of the post, so let's just assume we have this clear division. Yes, I'm talking about `DDD`, and often that is a good approach when you are growing the application. There are more links on the references.
{: .prompt-tip }


## Using the same Service

Once you identified the different bounded contexts, you need to `refactor the model` in order to isolate them. That really means, not sharing classes, tables and resources. 

> We need to isolate the query from the command so we can later freely change technology or optimize it as we wish.
{: .prompt-info }

After, the segregation we could get to a scenario like we have in the image bellow. 

![overview](/assets/img/diagrams/cqrs/first-split.excalidraw.png)

With this approach, you gain a lot of flexibility, to change the query patterns without interfering in the command patterns. The `dual write` problem for having multiple datastores in this case is not be much of an issue, because a blog post doesn't necessarily need to be updated in real time, so a simple chron job could fix any inconsistency. Needless to say, that you don't need to have two different datastores, I'm just illustrating that you could have. Yes, it is still a `Monolithic` approach, and still can face the issue when a bug on the writer side can bring down the readers application, but depending on the size of your customer base is a good **cost/benefit**. Let's explore a more robust and expensive approach next.

##  Event sourcing

Event sourcing is a great way to implement `CQRS`, by publishing the events with the state of the entities we want to replicate, we can later use it in multiple places in completely different manners. I talk about different approaches and issues you will encounter when doing this data replication on this post, [Outbox Pattern](../outbox-pattern/).

![overview](/assets/img/diagrams/cqrs/cqrs-overview.excalidraw.png)

> Avoid SPOF in your architecture
{: .prompt-tip }

By using this approach there is a clear reduction on coupling, since the source of truth of `Article` entity doesn't know anymore who is using it and how is being used. Also, the single point of failure `SPOF` was removed, now if the writer part of the application is down, readers will not notice it, and the other way around as well. Depending on how we implement the topic, let's say if we use a `compacted kafka topic` we can always bring new applications to life and have the whole replication of the database available to them. 

A very important aspect of this approach is that now that we completely separated the concerns, we can have independent teams working on the different `bounded contexts`, without the need to interact constantly to each other. This is often neglected as an advantage, but IMHO when a company is looking to scale productivity and paralyze work streams this is the way to go. 

> There ain't no such thing as a free lunch
{: .prompt-warning }

This is all wonderful, so *what is the catch?* This approach comes with a cost, first a monetary cost of the infrastructure of corse, you need a robust structure of a kafka cluster with replication and so forth to ensure not loosing events, more development on both sides, extra services, extra CI/CD concerns, evolution of the model will be more complex as it will need to be backwards compatible, etc. 

There is also the cost of the eventual consistency, that your system might not be able to pay. Eventual consistency in short means, that your system might be in an inconsistent state for some time until it gets to a consistent state, `after a writer changes an Article readers will still read an old article for some time until the data is propagated`. This is something that you have to accept when working with events, I even dare to say when working with most distributed systems.

So after all this considerations would you say it is worth it to implement this approach? That would depend you your scale, but definitely is a very powerful tool and can scale your system quite a lot.

## Conclusion

`CQRS` is a technique to segregate the commands from the query it can be implemented in many different ways, but the main idea is that you can freely evolve your use cases and optimize them independently. How you would implement it depends a lot on the volume of requests your application has, but it is definitely a very powerful tool to scale your application.

What do you think? Is it something that would help your application to scale? Let me know!


## References
 - [Martin Fowler - CQRS](https://martinfowler.com/bliki/CQRS.html)
 - [Confluent - Event sourcing](https://developer.confluent.io/learn-kafka/event-sourcing/cqrs/)
 - [Alberto Brandolini - About Team Topologies and Context Mapping](https://blog.avanscoperta.it/2021/04/22/about-team-topologies-and-context-mapping/)
