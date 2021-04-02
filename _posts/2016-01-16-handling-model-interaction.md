---
layout: post
title: "Implementing DDD: Part 6 - Handling model interaction"
excerpt: "In this chapter we are looking into how we beautifully can interact between the WebApi controller and our domain model"
tags: [Domain Driven Design, Architecture, RavenDb, WebApi]
comments: false
---

## Why not handle model interaction in the web api handler?

When dealing with webapi we most often call our controllers something like productController, and then we have the methods names with get, post etc. So it can be hard to distinguish the controllers and the use case they serve. In order to combat this, we can make use of the commmand pattern, and thus encapsulationg our use case in a class. The upsides to this, to name a few, is maintainability, testability as well seperating the delivery mechanism from the use case.

## Making a resolveable command
Let's assume that we have a use case called `CreateUser`. The create user action will create a user in our database, and it will return a `SuccessfulResult` if the user was created.

<br />
The command pattern have been described in many lengths in other blog posts so i would'nt bother you with that. However we can assume that each of our command will need access to the database and therefore will, start constructing our base command by parsing in the current session.

``` csharp
public abstract class Command<T>
{
    protected readonly IAsyncDocumentSession Session;

    protected Command(IAsyncDocumentSession session)
    {
        Session = session;
    }

    public abstract Task<T> Execute();
}
```

Now we can start constructing our `CreateUserCommmand`. Here we just demonstrating how to create a simple user. The fancy implementation will be up to you to write :)

``` csharp
public class CreateUserCommand : Command<SuccessfulResult>
{
    public string Username { get; set; }

    public CreateUserCommand(IAsyncDocumentSession session) : base(session){}

    public async override Task<SuccessfulResult> Execute()
    {
        var user = new User(Username);
        await Session.StoreAsync(user);

        var response = new SuccessfulResult());
        return response;
    }
}

public class SuccessfulResult {}
```

Remember to register the newly created commands in our `DomainInstaller`.
``` csharp
Classes.FromThisAssembly()
    .BasedOn(typeof (Command<>))
    .WithServiceBase()
    .LifestylePerWebRequest(),
```

## Connecting Commands with WebApi Controller

Now that we have a perfectly good `CreateUserCommand`, lets put it to good use. Our goal is to create a UserController which executes our command.

``` csharp
[RoutePrefix("api")]
public class UserController : RavenApiController
{
    [Route("CreateUser")]
    [HttpPost]
    public async Task<OkResult> TestA(UserDto dto)
    {
        var response = await CommandFactory<CreateUserCommand, SuccessfulResult>(createUserCommand =>
        {
            createUserCommand.Username = dto.Username;
        }).Execute();

        return Ok();
    }
}
```

We have quite abit going on here, first we'll notice that our Controller is inherited from `RavenApiController` instead of the normal `ApiController`, secondly we are resolving our Command through a `CommandFactory` method.

The `RavenApiController` is a custom ApiController that makes `CommandFactory<TInput, TRespons>` available in our inherited controllers. This method will resolve and return the requested command. When using the commandfactory we'll get and instance of the command parsed in as an `Action<TInput>` parameter. This will allow us to access all the public properties, and thus create a nice way of parsing inputs from our webapi function to our command.

<br/>
The last thing we'll do before returning OK, is executing our command.

## Creating the RavenApiController
The implemetation for the our `RavenApiController`is pretty simple. Basically what we do is that we are enabling the developer to resolve the requested command. However we need to pay attention to memory management. The rule of thumb, is that whenever you resolve something from the container we must manually release it again. With this in mind, we need have a `ISet<object>` where we can collect our resolved commands. When our controller is disposed, we make sure that be release the things we have used. 

``` csharp
public class RavenApiController : ApiController
{
    private readonly ISet<object> objToBeDisposed;

    public RavenApiController()
    {
        objToBeDisposed = new HashSet<object>();
    } 

    protected Command<TResonse> CommandFactory<TInput, TResonse>(Action<TInput> action) where TInput : Command<TResonse>
    {
        var context = HttpContext.Current.GetOwinContext();
        var container = context.Get<IWindsorContainer>("windsor-container");

        var cmd = container.Resolve<Command<TResonse>>();
        objToBeDisposed.Add(cmd);

        var input = cmd as TInput;

        if(input == null)
            throw new InvalidCastException($"Could not cast {typeof(Command<TResonse>)} to {typeof(TInput)}");

        action(input);

        return cmd;
    }

    protected override void Dispose(bool disposing)
    {
        var context = HttpContext.Current.GetOwinContext();
        var container = context.Get<IWindsorContainer>("windsor-container");

        foreach (var obj in objToBeDisposed)
        {
            container.Release(obj);
        }

        base.Dispose(disposing);
    }
}
```

## Conclusion
We have succesfully split the delivery mechanism from our usecases and we have an beatifully created a context aware Commands, which easilty can be tested and maintained. 

<br />
This is part 6 in the implementing DDD Series, be sure to checkout the project page on github
[here](https://github.com/mclausen/Raven.DDD.SampleWebservice)