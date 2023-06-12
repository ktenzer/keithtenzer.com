--- 
layout: single
title:  "Temporal Fundamentals Part I: Basics"
categories:
- Temporal
tags:
- Workflows
- Activities
- Queues
- Determinism
- Timeouts
---

![Temporal](/assets/2022-08-15/logo-temporal-with-copy.svg)
## Overview
This is a five part series focused on Temporal fundamentals. It represents, in my words, what I have learned and the short TLDR version.  

- [Temporal Fundamentals Part I: Basics](https://keithtenzer.com/temporal/Temporal_Fundamentals_Basics)
- Temporal Fundamentals Part II: Concepts
- Temporal Fundamentals Part III: Timeouts
- Temporal Fundamentals Part IV: Workflows
- Temporal Fundamentals Part V: Workers

## The Problem
In today's world we use APIs and those APIs are inherently unreliable. To perform higher level business processes or operations, we often need several APIs, which in turn are spread across many distributed and disparate systems. Any failure leads to an inconsistent state requiring recovery. While of course we can deal with this challenge,  building reliability into systems is more than a full time job for developers and we won't do it better than Temporal. 

Let's look at this problem in more detail, below is a very simple payment transaction involving just two APIs (withdraw and deposit).
```java
  public void Payment(
      String fromAccountId, String toAccountId, String referenceId, int amountCents) {
    account.withdraw(fromAccountId, referenceId, amountCents);
    account.deposit(toAccountId, referenceId, amountCents);
  }
```

If the deposit API fails an inconsistent state is reached as money is withdrawn, however no deposit can occur. To solve this problem, state must be maintained and persisted to a database. In addition retry and often timer mechanisms are required for handling failure recovery. Then of course it all needs to scale, right? The result is a very complex architecture to handle what was a simple API orchestration. Imagine the additional code to do all this at scale?

![Transfer without Temporal](/assets/2023-06-07/payment_wo_temporal.png)

What if you you didn't have to write all that extra code for handling failures? Guess what, you don't!
Temporal provides the solution, which is durable execution. By enabling durability for every function or API call, Temporal is able to greatly reduce complexity, allowing developers to simply orchestrate their APIs and focus on business logic without worrying about durability. Developers can write the exact code above and have it be durable without building the durability themselves.

![Transfer with Temporal](/assets/2023-06-07/payment_with_temporal.png)

## The Basics
Temporal enables a new programming model where application coordination and durability becomes a simple, easy to understand, workflow. To do this, Temporal provides SDKs in various programming languages, enabling developers to write applications as workflows, which interact with the Temporal server. The Temporal server handles coordination of tasks associated with workflow execution. Your workflow code and activity tasks are executed on a worker, built using the Temporal SDK. Workers run in your environment.

The temporal server can be self-hosted using the [opensource](https://github.com/temporalio/temporal) or consumed as a cloud SaaS via [Temporal Cloud](https://temporal.io/cloud).

## Temporal Features
Below is a list of just some of the capabilities enabled by using Temporal.

- Almost limitless scale
  - Temporal can scale to billions of concurrent workflows
- Durable Execution
  - Temporal provides workflow history where all events are persisted
- Resilience
  - Temporal replay will reconstruct your workflow state in the event of any infrastructure failure, picking up where it left off 
- Correctness
  - A workflow will eventually progress to completion regardless of failures
- Retries
  - Define retry policy with intervals and backoff coefficient
- Timers
  - Using both synchronous and asynchronous timers is trivial
- Heartbeats
  - Long running activities can heartbeat the Temporal server
- Schedules
  - Workflows can be seamlessly scheduled
- Interactive
  - Temporal workflows are interactive and can be signaled or queried
- Workflow Management
  - UI/CLI/APIs for managing workflows        

## Temporal Components
The main components in the Temporal ecosystem are: server, SDK, tooling and application.

### Temporal Server
As mentioned the Temporal server orchestrates workflows and provides workflow management. A very important point is that workflows are NOT executed on the Temporal Server. Instead they are executed by a worker which is polling the server for work. The Temporal server maintains state and correctness of a workflow in its event history. 

![Temporal Server](/assets/2023-06-07/temporal_server.png)

Temporal can handle billions of workflows, as such searching workflows is really important. The Temporal server provides standard and advanced visibility. Standard visibility allows you to search for workflows based on Status, WorkflowId, RunId, WorkflowType, Start Time and End Time. 

Advanced visibility, which uses Elasticsearch, lets you define custom attributes called search attributes, that can be set inside your workflow code using the Temporal SDK. This enables searching workflows by key/value pair, using a query.

The UI enables query or display of both standard and advanced visibility fields. Below BackupStatus is a custom search attribute, set in workflow code.
![Temporal Server](/assets/2023-06-07/visibility.png)

### Temporal SDK
The Temporal SDK enables developers to build a Temporal Application and interact with the Temporal server. The SDK supports many languages: Go, Java, Typescript, Python, .NET, PHP, Ruby and even other languages through the CoreSDK. The SDK provides additional tooling for debugging such as the workflow replayer. Each SDK also provides a rich set of samples so developers can get started quickly. The [Temporal developers guide](https://docs.temporal.io/dev-guide/) is a great starting point to building your first Temporal application.

### Temporal Tooling
Temporal provides both CLI and UI interfaces in addition to of course the SDK. The [cli](https://github.com/temporalio/cli) even includes a local development server which can run directly on your laptop. Simply follow the instructions on the linked Github repository for the cli and in 5 minutes you will be up and running!

### Temporal Application
A Temporal Application consists of a workflow starter and one or more workers. Workflows and activities are registered to workers which long poll the Temporal server on a specified task-queue. 

#### Workflow Starter
A starter is a simple program that starts a workflow. Starting a workflow is be done programmatically using the Temporal SDK or using the CLI.

Here we start a workflow called SampleWorkflow, provide a workflowId and a task queue.
```java
SampleWorkflow workflow = client.newWorkflowStub(
        SampleWorkflow.class,
        WorkflowOptions.newBuilder()
                .setWorkflowId("WORKFLOW_ID")
                .setTaskQueue("TASK_QUEUE")
                .build());

workflow.start();
```

Once a workflow is started, the Temporal server will begin progressing that workflow until completion. It will schedule tasks and match them to workers on the specified task-queue waiting for completion. While that is happening, it will ensure task failures or timeouts are handled according to the specifics defined in the workflow/activity configuration.

### Application Worker
A worker is a lightweight service built using the Temporal SDK. Workflows and activities are registered against a worker. The worker will then long poll against the Temporal server for tasks to execute against a pre-defined task-queue.

Here we create a worker, register our workflow and activity definitions.
```java
WorkerFactory factory = WorkerFactory.newInstance(client);
Worker worker = factory.newWorker(TASK_QUEUE);

worker.registerWorkflowImplementationTypes(SampleWorkflow.SampleWorkflowImpl.class);
worker.registerActivitiesImplementations(new SampleWorkflow.SampleActivityImpl());

factory.start();
```

## Summary
In this article we discussed the Temporal fundamental basics. We explained the challenge with distributed services, explaining how Temporal provides an elegant solution,  durable execution, enabling our code to run reliably by default. We looked at some of Temporal's capabilities and features. Finally we discussed the components in the Temporal ecosystem such as the Temporal server, SDK, tooling and application.

(c) 2023 Keith Tenzer




