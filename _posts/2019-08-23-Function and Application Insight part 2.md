---
layout: post
title: Azure Functions logs in Application Insights part 2 
author: Mattias Lögdberg
tags: [Azure Functions, Integration,Serverless, Application Insights]
categories: [Azure Functions, Application Insights]
image: 
description: 
permalink: /2019/02/functions/understandhowlogsinaipart2
---

In part one I covered the ILogger interface and what it's providing for us, but sometimes we want more control of our logging, enrich oru logging or we have allready implemented alot of logging with *TelemtryClient* and just want to connect the logging to our **end-to-end** experience.

We use the same Scenario, a message is recieve via API Management, sent to a Azure Function that publishes the message on a topic, a second Azure Function reads the published message and publishes the message on a second topic, a third Azure Function recieves the message and sends the message over http to a Logic App that represents a "3:e party system".

![Scenario image](/assets/uploads/2019/08/functionsAi-scenario.png)

Read more about the scenario background and the standard *ILogger* interface in previous post [Azure Functions logs in Application Insights](http://mlogdberg.com/2019/02/functions/understandhowlogsinai).


## Connecting custom logs from TelemetryClient
If we need more custom logging options than the provided in the standard *ILogger* provides or we have alot of logging today with *TelemetryClient* that is not connected to the endo to end scenario. We need attach these to our end-to-end logging experience via the **TelemtryClient** to take advantage of the more advanced logging features.

In order to achieve this we need to add some context properties to our instance, here is one way to do this via Service Bus, here we recieve the Message object.

From the Message object we can extract the Activity, this is an extension found in the *Microsoft.Azure.ServiceBus.Diagnostics* namespace. From the result of this operation we can get the *RootId* and *ParentId* wich is the values for  the current Operation Id and the parent id. [Read more on how the Operation id is built and how the hierarchy is designed](https://github.com/dotnet/corefx/blob/master/src/System.Diagnostics.DiagnosticSource/src/HierarchicalRequestId.md)

```csharp
public string Run([ServiceBusTrigger("topic1", "one", Connection = "servicebusConnection")]Message message, ILogger log)
{
    string response = "";
    var activity = message.ExtractActivity();

    telemetryClient.Context.Operation.Id = activity.RootId;
    telemetryClient.Context.Operation.ParentId = activity.ParentId;

    telemetryClient.TrackTrace("Received message");
```

This can also be done from a HTTP call [read more](https://docs.microsoft.com/en-us/azure/azure-monitor/app/custom-operations-tracking#http-request-in-owin-self-hosted-app).

```csharp
// If there is a Request-Id received from the upstream service, set the telemetry context accordingly.
if (context.Request.Headers.ContainsKey("Request-Id"))
{
    var requestId = context.Request.Headers.Get("Request-Id");
    // Get the operation ID from the Request-Id (if you follow the HTTP Protocol for Correlation).
    telemetryClient.Context.Operation.Id = GetOperationId(requestId);
    telemetryClient.Context.Operation.ParentId = requestId;
}
	
//GetOperationId method
public static string GetOperationId(string id)
{
    // Returns the root ID from the '|' to the first '.' if any.
    int rootEnd = id.IndexOf('.');
    if (rootEnd < 0)
        rootEnd = id.Length;

    int rootStart = id[0] == '|' ? 1 : 0;
    return id.Substring(rootStart, rootEnd - rootStart);
}
```


Now we will also add this small line of code that trackes an event.

```csharp
var evt = new EventTelemetry("Function called");
evt.Context.User.Id = "dummyUser";
telemetryClient.TrackEvent(evt);
```

And this can now be found in the end-to-end tracing.

![Event in Overview](/assets/uploads/2019/08/functionsAi-customEventOverview.png)
![Event in Telemetrylist](/assets/uploads/2019/08/functionsAi-endtoendoevent.png)

## Adding an Extra "Operation"
This is probobly my favorit thing in all of this, when processing messages some parts of the process are bigger and/or more important to understand, this could be a complex transformation or just a set of steps that we want to group.
Then we can use the *StartOperation* in *TelemetryClient*, this will start the scope for the *Operation* and it will be open until we execute the operation *StopOperation* it has a status and we can set some properties and get execution time of the process.

An Operation is either a *DependencyOperation*, intended to mark dependency to other resources or a *RequestOperation* meant for marking a creation of a request.
I will use a DependencyOperation in my case since it's the most alike operation for my purpose, I use it to mark that a more complex logic has been executed in my function and what the result of it was.

I use this in several ways one is when I process alot of items in my Function to mark that they are executed, like processing a record in a list, another is when performing a complex transformation and so on.

In my example bellow I'll demonstrate the usage of an operation by doing a simple for loop and simulating processing of a line and adding some valuable metadata. As a tip set the *Data* property to the object that is going to be processed to easier understand what data is processed.

```csharp
for (int i = 0; i < 10; i++)
{
    var op = telemetryClient.StartOperation<DependencyTelemetry>(string.Format("Function2_line_{0}", i.ToString()));

    op.Telemetry.Data = string.Format("Processing {0} and random guid {1}", i.ToString(), Guid.NewGuid().ToString());
    op.Telemetry.Type = "LineProcessor";
    op.Telemetry.Success = i < 9;
    op.Telemetry.Sequence = i.ToString();
    op.Telemetry.Context.GlobalProperties.Add("LineID", i.ToString());
	
	//do some heavy work!
	
    telemetryClient.StopOperation(op);                    
}
```

This will result in a mutch more detailed log in the **End-To-End** overview

![Operation in Overview](/assets/uploads/2019/08/functionsAi-endtoendwithoperation.png)

Custom properties is easily set on each operation for traceability or information, bellow is how we can add **LineID** as a custom property.

```csharp
op.Telemetry.Context.GlobalProperties.Add("LineID", i.ToString());
```

![Operation and custom Property](/assets/uploads/2019/08/functionsAi-operations-custom-properties.png)


We can also set the Operation to have failed even if we continued the run, good for highlight parts of failure while still needing to complete the rest of the run.

![Operation and custom Property](/assets/uploads/2019/08/functionsAi-operationfailed.png)

Then we can find it under failures and from there start working on how to fix it.

![Operation and failed operations](/assets/uploads/2019/08/functionsAi-operation-failures.png)

Read more: 
* [https://docs.microsoft.com/en-us/azure/azure-monitor/app/custom-operations-tracking](https://docs.microsoft.com/en-us/azure/azure-monitor/app/custom-operations-tracking)
* [https://docs.microsoft.com/en-us/azure/azure-monitor/app/api-custom-events-metrics](https://docs.microsoft.com/en-us/azure/azure-monitor/app/api-custom-events-metrics)

## Summary:
In this second part we have covered more advanced logging with TelemetryClient or to connect your current TelemetryClient logging to the End-To-End experience provided by Application Insights.

The StartOperation is really powerfull to highlight what is happening, but I would like to have a third option for an Operation since using the Dependency sounds wrong when it's a section of the code in the same process, but it works!

This post is about how things can be done and hopefully this can be a guidance in creating a better logging experience, we want to avoid the developers "let me just attach my debugger" response to what is the problem with the flow running in production.