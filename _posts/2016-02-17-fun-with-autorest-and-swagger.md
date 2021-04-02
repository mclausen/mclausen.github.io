---
layout: post
title: "Fun with AutoRest and Swagger"
excerpt: "Creating typed client with Autorest swagger"
tags: [Rest, WebApi, Autorest]
comments: false
---

Generally when dealing with alot of microservices we typically dont want to create typed clients because it can be hard to manage versions and dependencies, and i would generally advice you to
roll your own dto convertion logic. But every once in a while you come across places where it makes sense to create a strongly typed clients.

<br />
Look and behold...

## AutoRest
AutoRest is a tool which can turn swagger documentation into strongly typed clients! For many reasons this is extremely cool. To name a few

* Create clients with a run of a command line tool
* You dont waste time writing parsers
* Easily update your client whenever the endpoints has been updated

<br />
The only thing we have to do is to install Swagger and AutoRest. So fire up your favorite visual studio package management console and go:
    `install-package swashbuckle` and
    `Install-Package AutoRest`

### Automation automation automation

Often when dealing with commands alot, time we'll be spend typing in the arguments, getting the correct syntax, and when you finally run the command it gives your errors... See my point?

Therefore i have put together a litte powershell script that invokes autorest, and puts the typed client a output directory.

``` powershell
$swaggerEndpoint = "http://localhost:54403/swagger/docs/v1"
$outputDirectory = ".\server\proxies"
$autoRestRelativePath = ".\server\packages\autorest.0.14.0\tools\AutoRest.exe"
& $autoRestRelativePath -Input $swaggerEndpoint -Namespace AutoRestTest -OutputDirectory $outputDirectory -CodeGenerator CSharp
```

I've created a small example which can be found on [Github](https://github.com/mclausen/AutoRestExample)