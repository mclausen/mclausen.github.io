---
layout: post
title: "Implementing DDD: Part 5 - RavenDb Unit of work"
excerpt: "Handle all your database work in a single unit of work"
tags: [Domain Driven Design, Architecture, RavenDb]
comments: false
---

## RavenDb Owin Unit of Work

This series follow a sammple application which can be found here [https://github.com/mclausen/Raven.DDD.SampleWebservice](https://github.com/mclausen/Raven.DDD.SampleWebservice)

<br/>
Picking up from the previous post were implemented the `OwinDomainEventPublisher` to collect all Domain events being thrown when acting on our model. At the end of our unit of work we'll be dispatching the collected domain events.

<br/>
So to get started started we'll be implementing the UOW as a part of our `OWIN` pipeline. Go ahead and create a new class called UnitOfWork and let it derive from `OwinMiddleware`.

<br/>
The first thing we are going to do when overriding the Invoke method is to add the container to our Owin Context, this will come handy later when need to access the container from other places in the application without relying on a static method call to the container.
We then new up our `OwinDomainEventPublisher` and pass it along to `DomainEvents.Current` which static accessable way to throw domain events.

<br/>
Next we are going to Resolve `IAsyncDocumentSession` from  the container. This is the session that we'll be using throughout our entire session. 

<br/>
After we have initiated our session we'll be stepping into the next step in the owin pipeling by calling `await Next.Invoke(context);`. For now we are going to assume that it will be our web api handler and thus acting on our domain model.

<br/>
Last thing we need to take care of before saving in the session, is dispatching all the collected domain events.

``` csharp
public class RavenDbUnitOfWork : OwinMiddleware
{
    private readonly IWindsorContainer _container;

    public RavenDbUnitOfWork(OwinMiddleware next, IWindsorContainer container) : base(next)
    {
        _container = container;

    }

    public override async Task Invoke(IOwinContext context)
    {
        context.Set("windsor-container", _container);
        var domainEventPublisher = new OwinDomainEventPublisher(context);
        DomainEvents.Current = domainEventPublisher;

        var session = _container.Resolve<IAsyncDocumentSession>();

        await Next.Invoke(context);

        await DispatchDomainEvents(domainEventPublisher);

        await session.SaveChangesAsync();

        session.Dispose();
        _container.Release(session);
        
    }
}

```

## Dispatching domain events

Before we are saving the UOW we are dispaching all our domain events we have collected. We simply resolve all our handlers that implements the `ISubscribeTo<>` interface and the use reflection to invoke thier Handle method. Easy peasy nice and easy :)

``` csharp
private async Task DispatchDomainEvents(OwinDomainEventPublisher domainEventPublisher)
{
    foreach (var domainEvent in domainEventPublisher.CollectedEvents)
    {
        var typeParam = domainEvent.GetType();
        var type = typeof(ISubscribeTo<>).MakeGenericType(typeParam);
        var subscribers = _container.ResolveAll(type);

        foreach (var domainEventSubscriber in subscribers)
        {
            await (Task) domainEventSubscriber
                .GetType()
                .GetMethod("Handle")
                .Invoke(domainEventSubscriber, new object[] {domainEvent});

            _container.Release(domainEventSubscriber);
        }
    }
}
```

## Hooking up our unit of work into the owin pipeline

Controlling the owin pipeline is easy as pie. When an incomming request hits the server, it will take go through the step in the order they're defined and since the first thing we want is to initiate our unit of work, go ahead and add  `app.Use<RavenDbUnitOfWork>(Global.Container);` as the first step to the startup file. So it will look something like this.

``` csharp
public class Startup
{
    public void Configuration(IAppBuilder app)
    {
        var httpConfiguration = new HttpConfiguration();
        httpConfiguration.MapHttpAttributeRoutes();
        httpConfiguration.EnsureInitialized();

        app.Use<RavenDbUnitOfWork>(Global.Container);
        app.UseWebApi(httpConfiguration);
    }
}
´´´