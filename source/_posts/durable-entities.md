---
title: Getting started with Durable Entities
tags:
  - Azure Functions
  - Durable Entities
  - Durable Functions
  - Azure
  - Functions
  - Serverless
  - Serverless Functions
  - Actors
  - Function Actors
date: 2020-03-31 09:23:54
---


Entity functions define operations for reading and updating small pieces of state, known as durable entities. Like orchestrator functions, durable entities are functions with a special trigger type, the entity trigger. Unlike orchestrator functions, entity functions manage the state of an entity explicitly, rather than implicitly representing state via control flow. Entities provide a means for scaling out applications by distributing the work across many entities, each with a modestly sized state.

Source: [Microsoft](https://docs.microsoft.com/nl-nl/azure/azure-functions/durable/durable-functions-entities?tabs=csharp)

In this post, we will have a look at how you can get started with Durable Entities. We will go through the concepts that you need to understand and how to implement them. For this article, we will be building an application for a hospital to check if a hospital bed is still occupied or not. A big plus of using Azure Durable Entities is that we don't need to worry about the plumbing. The framework abstracts most of the state persistence tasks for us. So let's see how we can take advantage of this.

To get started with Durable Entities, you need some basic knowledge about Azure functions and the concept of durable functions. A durable function uses a 3 part system. First, you have the HTTP client function that receives the external call. This function is always stateless because it just gets the request and validates it. After that, it creates an Orchestrator that handles the request flow. Once the Orchestrator is created, he can start to create Activities that do the actual work here. So just to quickly recap, we have:

- Http Client Function(Stateless) - Receives the request, validates it, Sets up the Orchestrator.
- Orchestrator - Has the flow logic and triggers Activities.
- Activities - This has the actual compute logic and does the work.

For more information on these topics please see [here](https://docs.microsoft.com/nl-nl/azure/azure-functions).

## What is an Entity?

An Entity is what u expect it to be. If you worked with Entity-Framework or DDD before it should be a familiar term. An entity is a class consisting of properties that represents a set of data in some kind of storage.

In our case, the state of the Entity is persisted in Azure Table Storage by the framework. Entities in Durable Entities are a bit different because they are like small microservices (Actors) that have methods on them that you can call. To call an Entity, we need to know its `Entity ID`. This Id uniquely identifies the Entity. An Entity-Id consists of a name and a key that together form a unique string.

There are also other properties on an Entity like:

**Operation name**, this would be the name of an operation that can be performed. For example `AssignBedAsync`.

**Operation Input**, These are parameters that can be provided with the operation to perform. These parameters are optional. For example the name of the person that will occupy the bed.

**Scheduled time**, This optional parameter indicates the specific time to run. Operations can be scheduled to run in the future using this parameter.

A very simple Entity looks like this:

```csharp
public class BedEntity
{
    public string BedNumber { get; set; }
    public bool IsOccupied { get; set; }
    
    private readonly ILogger _logger;

    public BedEntity(ILogger logger, string bedNumber)
    {
        _logger = logger;
        BedNumber = bedNumber;
    }
    
    [FunctionName(nameof(BedEntity))]
    public static async Task HandleEntityOperation(
        [EntityTrigger] IDurableEntityContext context,
        ILogger logger)
    {
        await context.DispatchAsync<BedEntity>(logger, context.EntityKey);
    }
    
    public Task<bool> IsOccupiedBedAsync()
    {
        var IsOccupied = BedNumber == "123";
        return Task.FromResult(IsOccupied);
    }
    
    public Task AssignBedAsync(string bedNumber)
    {
        IsOccupied = true;
        return Task.CompletedTask;
    }
}
```

If we look at the above code, you might be thinking, but how and where does the call to the database take place to store the `BedEntity` after it's modified? This is all abstracted by the framework. We do not need to worry about it. The method `AssignBedAsync` changes the entity's state asynchronously, and then the framework does the rest. The function `HandleEntityOperation` allows us to call operations on this entity from the client function or orchestrator.

## Using the Entity

Now let's have a look at a simple client function that would make use of this `BedEntity`:

```csharp
public static class BedManager
{
    [FunctionName(nameof(AssignBed))]
    public static async Task<IActionResult> AssignBed(
        [HttpTrigger(AuthorizationLevel.Function, "get")]
        HttpRequest req,
        [DurableClient] IDurableEntityClient durableEntityClient,
        ILogger log)
    {
        var bedNumber = req.Query["bedNumber"];
        var entityId = new EntityId(nameof(BedEntity),bedNumber);
        
        log.LogInformation($"Received request for bed {bedNumber}");
        
        var bedEntity = await durableEntityClient.ReadEntityStateAsync<BedEntity>(entityId);
        if (bedEntity.EntityExists && bedEntity.EntityState.IsOccupied)
        {
            return new BadRequestObjectResult("Bed is already occupied.");
        }
        
        await durableEntityClient.SignalEntityAsync(entityId, nameof(BedEntity.AssignBedAsync));

        return new OkObjectResult($"Bed {bedNumber} has been set to occupied.");
    }
}
```

If we now run this client Function and send a request to it (bedNumber = 123) with a breakpoint on the logline we see the entityId being constructed as `@BedEntity@123`. The framework will use this id to get the current state if an entity is found using this id. If an entity is found using the `ReadEntityStateAsync` method on the `IDurableEntityClient` we check if the bed is also occupied. If that is the case we cannot assign it and return a `BadRequest`. If no BedEntity could be found with this id or the bed is not occupied we signal the entity to assign a bed with the provided `bedNumber` using the `SignalEntityAsync` method. This method gets the entity and triggers the `AssignBedAsync` call which changes the state. The framework then persists the changes in the entity state back to table storage.

But what happens in the underlying framework if we call `SignalEntityAsync`? What happens is that the framework starts up a virtual actor, as Microsoft calls it. It's a function that acts on our request to process information regarding the entity's state. The framework uses the existing durable queues that we know from the already existing durable functions framework built on which this is built on.

Durable entities are in no way replacing the previously existing durable functions. They are an addition to the existing durable function framework to help you to get rid of some of the plumbing work that was still there. You can use all the parts that were already there in combination with this new addition. 

![Durable Function Types](images/durable-entities/functiontypes.png)

One of the key parts that Entities resolve is if you would send a lot of `AssignBed` requests the framework would process them for the entity there are requesting one by one. This means you don't need to write locks or checks to validate the concurrency. This is all handled by the framework.

In our sample, we only used the Client function and the Entity to get the job done. If you need more flexibility you could easily add an Orchestrator function that we know from the existing durable functions to the mix that handles more complex scenarios.

The complete code for the example shown previously can be found [here](https://github.com/fschaal/Durable-Entities).