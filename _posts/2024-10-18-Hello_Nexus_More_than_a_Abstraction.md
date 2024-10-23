--- 
layout: single
title:  "Hello Nexus: More than a Abstraction"
categories:
- Temporal
tags:
- Temporal
- Durable Execution
- Replay
- Determinism
- Workflow Orchestration
- Nexus
---

![Temporal](/assets/2022-08-15/logo-temporal-with-copy.svg)
## Overview
Temporal announced Nexus during [Replay 2024](https://temporal.io/blog/unlock-new-possibilities-with-product-updates-from-replay-2024). Nexus has been a feature and capability in the works for several years. Without Nexus, Temporal developers didn't have a lot of options for executing workflows or workflow primitives across namespace boundaries. There was no easy way to provide an API contract, and as such Workflows became a somewhat, leaky abstraction. Futhermore, beyond activities which were sometime overkill, developers did not have a way to make simple service calls, durably.

## Nexus Feature
Nexus is a groundbreaking feature and will take Temporal to the next level. It will fundamentally change, and open up new ways to build Temporal-driven applications. Nexus effectively removes the last few limitations that existed in Temporal, while also added a new way to call simple services durably.

Nexus enables:
- Cross Namespace Communications
- API Service Contract for Workflows and Workflow primitives
- Ability to call simple services durably, where Workflow and Activity may have been overkill
- Better team or domain boundaries

Multiple teams can now build domain services, backed by Temporal Workflows, which can be consumed or interacted with, in a variety of ways, using Nexus endpoints.

## How Nexus Works
Nexus performs low-level RPC operations and provides an integrated SDK experience, enabling developers to create a handler and then invoke that handler from within a Workflow.

Once a Nexus endpoint receives a operation request, it will match it to a worker that implements the handler within the Namespace.

When creating a Nexus endpoint, provide a service name, Namespace and allowed Namespaces, which can call the endpoint. Optionally,  a description can be provided as markdown.

![Nexus Endpoint](/assets/2024-10-21/nexus_create.png)

## Nexus Examples
Nexus is used to call a Workflow, Workflow primitive (Signal/Update/Query) or even another simple service directly. This example shows how to create a Nexus endpoint for a Workflow, and execute that Workflow using the Nexus endpoint in the Go SDK.

### Creating Nexus Service
First, create a service for the Nexus endpoint. Nexus allows developers to define API contracts. The below snippet, shows how to create Nexus service and define service inputs or outputs.

```go
const ServiceName = "my-service"
const OperationName = "my-op"

type Input struct {
	Item  Item
}

type Output struct {
	Message string
}
```

### Creating Nexus Handler
Next, create a Nexus handler. There are two methods to use for creating a handler. For calling a Workflow use ```NewWorkflowRunOperation``` otherwise use ```NewSyncOperation```.

As of today, Nexus operations must return something and do not support nil as a return. The below snippet, shows how to create a handler to run a Workflow.

```go
var NexusWorkflowOperation = temporalnexus.NewWorkflowRunOperation(
	app.OperationName,
	workflows.MyWorkflow,
	func(ctx context.Context, input app.Input, soo nexus.StartOperationOptions) (client.StartWorkflowOptions, error) {
		return client.StartWorkflowOptions{ID: fmt.Sprintf("workflow-%v", input.Item.Id)}, nil
	},
)
```

### Registering Nexus with Worker
As with Temporal Workflows and activities, Nexus services must be registered with a worker. The below snippet, shows how to register a Nexus service with a worker. 

Note: you will also need to register any Workflows or Activities that the Nexus operation should trigger.

```go
service := nexus.NewService(app.ServiceName)
err = service.Register(handler.NexusWorkflowOperation)
if err != nil {
	log.Fatalln("Unable to register operations", err)
}
w.RegisterNexusService(service)
```

### Calling a Nexus from Workflow
Now that we have defined a Nexus service, handler and registered it with a worker, we can call it from a Workflow. The below snippet, shows how to execute a Nexus Operation and provide back an Operation Id.

```go
var op workflow.NexusOperationExecution
service := workflow.NewNexusClient(os.Getenv("NEXUS_ENDPOINT"), app.ServiceName)

nf := service.ExecuteOperation(ctx, app.OperationName, Input, workflow.NexusOperationOptions{})
f = nf

nf.GetNexusOperationExecution().Get(ctx, &op)
logger.Info(" Started Nexus Operation: " + op.OperationID)
```

A more detailed example can be seen in [Temporal Order Management Demo](https://github.com/temporal-sa/temporal-order-management-demo/tree/main/go/nexus).

## Summary 
In this article we were introduced to the new groundbreaking Temporal feature: Nexus. Nexus provides an API contract and cross-namespace support for Temporal Workflows, as well as Temporal primitives such as Signal, Update or even Query. We discussed some of the use cases behind Nexus and saw first hand, with examples, how to easily invoke not only Workflows, Workflow primitives but any service that requires durability, using Nexus.

(c) 2024 Keith Tenzer




