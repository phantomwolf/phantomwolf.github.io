---
layout: post
title: "go-kit tutorial"
date: 2018-03-27 11:03:00 +0800
categories: web-development
---
## Introduction
Go kit is a collection of Go (golang) packages (libraries) that help you build robust, reliable, maintainable microservices.

By the end of this tutorial, you'll get a small blog microservice which supports creating/updating/searching/deleting blog posts.

## Design
A typical go-kit microservice consists of the following layers(from top to bottom):

* Transport: Handling actual transport protocals(HTTP, gRPC).
* Endpoint: An endpoint is like an action/handler on a controller, containing safety and antifragile logic, and middlewares.
* Service: In Go kit, services are typically modeled as interfaces, and implementations of those interfaces contain the business logic.

The above 3 layers would make a functioning microservice, but real-world applications are usually more complicated. For example, a large project may have more layers:
