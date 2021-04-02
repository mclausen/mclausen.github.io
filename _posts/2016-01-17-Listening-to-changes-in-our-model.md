---
layout: post
title: "Implementing DDD: Part 7 - Listening to changes in our model"
excerpt: "More often than not we need to have the ui updated with the latest changes from our database. In this chapter we will look into that by using Castle windsor and RavenDb Changes api"
tags: [Domain Driven Design, Architecture, RavenDb, Castle windsor]
comments: false
---

As users of an application, we like to see whats currently in the database. However when dealing with many users who changes data all the time, we can run into issues where the data presented can be out of date.
We can easilly take care of this, with ravendb's changes api.

## RavenDb notifications
RavenDb's changes api allows us to hook into different events, comming from the ravenDb server. Without going into great depts for each listener here the two I find most useful.

* `ForDocumentsOfType` - Allows us to handle changes for a document in collection. This can be used to get updated for single documents. Lets say that you have a user who performs and action to document. Now you have two options three options show the updates to the user.
	1. You can just change the display value in the UI and hopefully the change will be succelful. This will show the change immidiatly howerever, if the changes are not succesful we need to fetch and updated document or roleback somehow.
	2. Secondly we have the possibility to return a the resulting document for the change, and update the ui accordingly.
	3. Thirdly we can use a document change listener and publish events to the ui using SignalR. This will actually solve our problems. First problem bieng if you have multiple users looking at the same object and you'll need to display live updates for that entity. Secondly you'dd have the possibility queue operations, and waiting till the operation ends to display changes, and then communicate progress for longer running processes.

* `ForIndex` - Allows us to handle changes in a index - Allows us to listen for changes to an index. The place were this comes in handy is when you want to display a list based on an index. In reason history i was envolved in a project where we had to display a list of trades depending on where in the pipeline it was, being it Created, Approved, AwaitingAction etc. Listening to the changes in the index was the perfect fit for this scenario.

<br />
For more details feel free to checkout the blog [everydayin.net](http://everydaylifein.net/nosql/ravendb/notifications-from-ravendb-server.html) which is run by a good friend of mine who describes all the listeners in great detail.

<br />
And ofcourse the official [documentation](https://ravendb.net/docs/article-page/3.0/csharp/client-api/changes/what-is-changes-api) for the changes api.

## Castle windsor startables facility
What Castle windsors `StartableFacility` Basically allows us to do, is to have a service that starts up the same time as the server. We can utillize this to hook into changes API and dispatch events from there. To get started go ahead and add the following changes to our `DomainInstaller`.

<br />
First we are telling Castle windsor to register the startable facility, and secondly we are registering our classes which implements the `IStartable` interfaces.

``` csharp
public void Install(IWindsorContainer container, IConfigurationStore store)
{
    container.AddFacility<StartableFacility>();

    container.Register(
        Classes.FromThisAssembly()
            .BasedOn<IStartable>()
            .WithServiceBase()
            .LifestyleSingleton()
        );
}
```

It's important to note that the our startables service will live outside our unit of work, so make sure to register them as Singletons.

## Bringing it all together
I've created a little sample that implements the `ForDocumentsInCollection` change listener. As soon as the service starts we'll connect and recieve changes from the RavenDb server. When changes appear they'll show up in our implementation of `IObserver<>`. Her we have the chance to examinating the change and push changes we migth have to the client.

``` csharp
public class DocumentChangedListener : IStartable
{
    private readonly IDocumentStore _documentStore;
    private IDisposable _subscription;

    public DocumentChangedListener(IDocumentStore documentStore)
    {
        _documentStore = documentStore;
    }

    public void Start()
    {
        _subscription = _documentStore
            .Changes()
            .ForDocumentsInCollection<TestAggregateRoot>()
            .Subscribe(new TestAggregrateRootDocumentObserver());

    }
    public void Stop()
    {
        _subscription?.Dispose();
    }
}

public class TestAggregrateRootDocumentObserver : IObserver<DocumentChangeNotification>
{
    public void OnNext(DocumentChangeNotification value)
    {
        // Push your changes to UI
        // (...) 
    }

    public void OnError(Exception error) { }

    public void OnCompleted() { }
}
```