---
layout: post
title: "Implementing DDD: Part 4 - Collecting Domain events"
excerpt: "Collecting Domain events in a unit of work "
tags: [Domain Driven Design, Architecture, RavenDb]
comments: true
---

## Collecting Domain events in a unit of work 

if you havn't already, go grab my sample application [https://github.com/mclausen/Raven.DDD.SampleWebservice](https://github.com/mclausen/Raven.DDD.SampleWebservice)

<br />
Before we can use DomainEvents from our model we need to implement the `IPublishDomainEvent` interface provided by `RavenDb.DDD.Core` package. The idea is, that during our current unit of work, we'll pickup any domain event being thrown by the model, and then dispatch each event after the main work has been done. Our domain event publisher could look like the sample below.

``` csharp
public class OwinDomainEventPublisher : IPublishDomainEvent
{
    private readonly IOwinContext _context;

    public OwinDomainEventPublisher(IOwinContext context)
    {
        _context = context;
        _context.Set("domain-events", new Queue<IDomainEvent>());
    }

    public IEnumerable<IDomainEvent> CollectedEvents => _context.Get<Queue<IDomainEvent>>("domain-events");

    public async Task Publish<TDomainEvent>(TDomainEvent domainEvent) where TDomainEvent : IDomainEvent
    {
        var queue =_context.Get<Queue<IDomainEvent>>("domain-events");
        await Task.Run(() => queue.Enqueue(domainEvent));
    }
}
```

<br />
Notice that we'll take a dependency on the `IOwinContext` so that we only store events in current request context

<br />
We can now directly hook up our new `IPublishDomainEvent` `DomainEvents.Current = new OwinDomainEventPublisher(context)` in our `RavenDbUnitOfWork`, which we'll get to in a second.

<br />
From our model, we can now publish domain events by calling `DomainEvents.Publish(new SomeDomainEvent())` from our model and then it will get picked up by `OwinDomainEventPublisher`. Next will look into how we can create our unit of work, and then dispatch our collected domain events