---
layout: post
title: "Implementing DDD: Part 2 - Project and References"
excerpt: "Getting it right from the start!"
tags: [Domain Driven Design, Architecture, RavenDb]
comments: true
---

##Creating a solid foundation
So, you have decided you wanted to go down the DDD path, you have discovered a bounded context and want to start writing some code, however you havn't set up a solution yet, now where do you start?

<br />
First you can start by cloning my sample webservice here [https://github.com/mclausen/Raven.DDD.SampleWebservice](https://github.com/mclausen/Raven.DDD.SampleWebservice)

<br />
Go ahead and create an empty ASP.NET Web application and two class libraries the, and call them something like #Domain#.Webservice, #Domain#.Infrastructure and #Domain#.Core, and ofcouse substitude the #Domain# with your domain. Take a look on the picture below. The names are just examples.

<figure class="half">
    <a href="/images/Ravendb-solutionpng"><img src="/images/Ravendb-solution.png"></a>
    <figcaption>RavenDb solution</figcaption>
</figure>

### References and basic classes
We are building the webservice pipeline from scratch using Owin and Katana, so if you are not familiar with these technologies we suggest you get aquainted [here](http://www.asp.net/aspnet/overview/owin-and-katana).

1. After you have done this, you should setup the references and third party libraries. Webservice should depend on Infrastructure and model, and Infrastructure should only depend on the model.

2. Now go ahead and add `RavenDb.DDD.Core` to core, infrastructure and for the webservice and add Castle.Windsor and `RavenDb.Client` to infrastructure and Webservice

3.	Next we need to setup the webservice. Go ahead and grab `Microsoft.AspNet.WebApi.Owin` and `Microsoft.Owin.	Host.SystemWeb` from nuget and install it on the webservice.

4. Next we need a owin Startup class to implement our start specifing the OWIN pipeline and make sure to add `[assembly: OwinStartup(typeof(*Namspace*.Startup))]` if you are hosting this on a IIS Service. 

``` csharp
public class Startup
{
    public void Configuration(IAppBuilder app)
    {
        var httpConfiguration = new HttpConfiguration();
        httpConfiguration.MapHttpAttributeRoutes();
        httpConfiguration.EnsureInitialized();

        app.UseWebApi(httpConfiguration);
    }
}
```

5. If the webservice doesn't already contain a `Global.asax.cs` file go ahead and add on.
In the global.axas.cs file we are going to initialize and disposing our windsor container like so.

``` csharp
public class Global : System.Web.HttpApplication
{
    public static IWindsorContainer Container { get; protected set; }

    protected void Application_Start(object sender, EventArgs e)
    {
        Container = new WindsorContainer();
        Container.Install(FromAssembly.This());
    }

    protected void Application_End(object sender, EventArgs e)
    {
        Container.Dispose();
    }
}
```

<br />
Next we are going to hook up domain events! Stay tuned