---
layout: post
title: CQRS
categories: [patterns]
tags: [CQRS,command-query-responsibility-segregation,scaling]
mermaid: true
---
# Command Query Responsibility Segregation [CQRS]

There are moments where the patterns used for querying the models of an application are far different from those used for managing its life cycle. We will go over an example on this post on when this happen and a couple of alternatives to how to improve your system scale on such situation.
## Background

You manage a system that is responsible for managing blog posts, a Content management system, CMS. This system has two main functionalities, it allows writers to create and manage content, CRUD, and also allows readers to consume  the final content. The system was in the beginning with a small customer base, therefore no reason to make the architecture overcomplicated, so it was designed as a single deployment artifact (maybe multiple instances) using a single database, as demonstrated on the image below.

![overview](/assets/img/diagrams/cqrs/simple-deployment.excalidraw.png)

## Problem

The simple RestFul approach is no longer being enough to handle all your use cases. As the use cases evolve and the patterns to access the blog posts change, you understand that the use cases for the writer are quite different from the use cases for the reader and they are interfering in one another. The company is also growing and you want to speed up the development. How to move on?

## 1. Different concerns

In this particular example is very easy to spot that we have two clear different concerns, for instance:

* Reader:
  * might want to use textual search to find the posts
  * might want have them pre-rendered on the backend stead of sending Json to the front end to improve customer experience
  * reference to custom tags
* Writer:
  * might want to be able to store draft version of the posts
  * might want to schedule a publish time

There are scientific ways of find these bounded contexts, you could use for instance an [`Event Storming`](https://www.eventstorming.com/) session. This is not the focus of the post, so let's just assume we have this clear division.

## 2. Splitting the model

If we were using a same schema first, is fair to assume that we would have a mapping similar to what we have below, where the entities are mostly normalized, that means that we are reusing the fields for `Article`, in both contexts.

The first point to notice is that features like having draft changes starts to become challenging, you would need to create another entity for draft and sync it once in a while. Pre-rendered version and textual search don't go well with constant changes, so if your pattern for changes start to grow it will impact how those features perform.

That's exactly where `Command Query Responsibility Segregation` get's in the picture, we introduce different `models` that can fit better for the  patterns fo `command`, in other words all the manipulation of the entity, and the `query` different patterns for reading and accessing the end result.

Let's start by segregating the domain to separate the things that makes sense for each of the contexts.


