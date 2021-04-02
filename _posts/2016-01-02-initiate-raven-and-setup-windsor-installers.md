---
layout: post
title: "Implementing DDD: Part 3 - Initiate RavenDb and setup windsor Installers"
excerpt: "Hookup container and initialize ravendb"
tags: [Domain Driven Design, Architecture, RavenDb, Castle Windsor]
comments: true
---
###Initiate RavenDb and setup windsor Installers

Castle Windsor have a neat feature called installers where we collect, register and configure our dependencies lifestyle. We can utilize installers to have a central place where initialize our connection to RavenDb.

<br/>
First thing we need is to tell Caste Windsor to look for installers in a given assemebly. We configured our container in `global.asax`, so go ahead and add `Container.Install(FromAssembly.This());` just after we initialize our container.

<br/>
Next we need to create our RavenDb Installer like the example below.

``` csharp
public class RavenDbInstaller : IWindsorInstaller
{
    public void Install(IWindsorContainer container, IConfigurationStore store)
    {
        container.Register(Component.For<IDocumentStore>()
            .UsingFactoryMethod(CreateDocumentStore)
            .LifestyleSingleton()
            .OnDestroy(x => x.Dispose()));

        container.Register(Component.For<IAsyncDocumentSession>()
            .UsingFactoryMethod(CreateSession)
            .LifestylePerWebRequest());
    }

    static IDocumentStore CreateDocumentStore()
    {
        var documentStore = new DocumentStore
        {
            Url = "http://localhost:8080",
            DefaultDatabase = "RavenDDDStore"
        };
        
        documentStore.Initialize(ensureDatabaseExists: true);
        IndexCreation.CreateIndexes(typeof (TestAggregateRootIndex).Assembly, documentStore);

        return documentStore;
    }

    private IAsyncDocumentSession CreateSession(IKernel input)
    {
        var store = input.Resolve<IDocumentStore>();
        var session = store.OpenAsyncSession();
        return session;
    }
}
```

<br/>
We only need one `DocumentStore` for our main application so make sure that we Register the lifeStyle as a singleton, meaning that the instance that we create will only be called on time at startup. Note that we are also initializing our Indexes at the same time, so if you have created or changed an index, this will automatically be updated when you run the application.

### DomainInstaller
We also have a couple of domain services to install, so go ahead and create another installer. 

<br/>
Here is the snippet
``` csharp
public class DomainInstallerInstaller : IWindsorInstaller
{
    public void Install(IWindsorContainer container, IConfigurationStore store)
    {
        container.Register(
            //Classes.FromThisAssembly()
            //    .BasedOn(typeof (Command<>))
            //    .WithServiceBase()
            //    .LifestylePerWebRequest(),

            Classes.FromAssembly(typeof(SomeDomainEventSubscriber).Assembly)
                .BasedOn(typeof(ISubscribeTo<>))
                .WithServiceBase()
                .LifestylePerWebRequest()
            );
    }
}
```

This will register all our DomainEvent Subscribers from our insfrastruture layer. So if you havnt already, go ahead and substitude the `SomeDomainEventSubscriber` with a random class from your infrastructure layer

<br/>

### Conclusion
Now we should have ravendb client running and registered with our container, so that everytime we request `IAsyncDocumentSession`, we'll get fresh session. However this is not what we want and therefore we'll have to implement a unit of work using owin pipline. We'll get to this in a later post.

<br/>
Next up, we'll look into how we can react on domain events being published :)
