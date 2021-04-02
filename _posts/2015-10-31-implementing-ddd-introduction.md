---
layout: post
title: "Implementing DDD: Part 1 - The grand overview"
excerpt: "I'll start by introducing a sample project in which we are going to design and implement the basic architecture for an application using DDD"
tags: [Domain Driven Design, Architecture, RavenDb]
comments: false
---

# Introduction
Over the last couple of years I've been evolved in designing and implementing applications using domain driven design. I didnt come without a cost, but in the end we managed to find some pattern and a architectural style that helped us immensely.

<br />
Our focus for this series is to build the scaffolding serverside architecture, and paving way for developers stay focused on business objectives instead of [Yak shaving](http://www.hanselman.com/blog/YakShavingDefinedIllGetThatDoneAsSoonAsIShaveThisYak.aspx "Hanselsman's definition of yak shaving")

<br />
We are going to cover quite abit, so lets get started!

---

# Architectural overview

We start out by defining the the concepts in our architecture the buttom up
1. Core  
2. Infrastructure  
3. Host

<br />
These guys is the three layers in our Onion/hexagonal architectures where the infrastructure layer has a dependency on core, and host has a dependency on both Infrastructure and core.

<br />
This stack is ment to be pr. bounded context, where one application clearly defines in business area. This however can be scalled and it can work beatifully together in a destributed environment. We will be covering this as well, but we have stretch to go.

<figure class="half">
    <a href="/images/hexagonal-architecture.png"><img src="/images/hexagonal-architecture.png"></a>
    <figcaption>Hexagonal architecture that we are going to build</figcaption>
</figure>

<br />
We will cover all three layers extensively in the upcomming posts but here is a brief overview

## Core
This project will include All of our aggregate roots, value objects. And it is here that we are going to define our businesslogic by creating higly maintainable Aggregate roots and by any means avoid creating an [anemic domain model](https://en.wikipedia.org/wiki/Anemic_domain_model).

## Infrastructure
In here we are going to deal Domain event thrown from the core, implementing domain services, and deal with RavenDb and Rebus message bus.

## Host
The host is simply how are we going to run the application, is it throught a windows- or web-service or even a third option. 

### Technologies

* [RavenDb](http://ravendb.net/) - We have had alot of success implementing applications using Ravendb, and for that particular reason we are going to do the same!
* [Rebus](http://mookid.dk/oncode/rebus) - My favorite message bus!
* [Castle Windsor](http://www.castleproject.org/) - For all our ioc needs.