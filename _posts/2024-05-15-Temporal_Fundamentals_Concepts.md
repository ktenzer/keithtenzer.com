--- 
layout: single
title:  "Temporal Fundamentals Part II: Concepts"
categories:
- Temporal
tags:
- Temporal
- Durable Execution
- Replay
- Determinism
- Workflow Orchestration
- Workflow as Code
- Concepts
---

![Temporal](/assets/2022-08-15/logo-temporal-with-copy.svg)
## Overview
This is a five part series focused on Temporal fundamentals. It represents, in my words, what I have learned along the way and what I would've like to know on day one.

- [Temporal Fundamentals Part I: Basics](https://keithtenzer.com/temporal/Temporal_Fundamentals_Basics)
- [Temporal Fundamentals Part II: Concepts](https://keithtenzer.com/temporal/Temporal_Fundamentals_Concepts/)
- [Temporal Fundamentals Part III: Timeouts](https://keithtenzer.com/temporal/Temporal_Fundamentals_Timeouts/)
- Temporal Fundamentals Part IV: Workflows
- Temporal Fundamentals Part V: Workers

## Workflows as Code
In Temporal, workflows are code. There are no DSLs, DAGs, YAML, JSON or any other form of application domain specific language. You write your workflows in the programming language of your choice: Java, Typescript, Cotlin, Go, .NET, Python or PHP. This contrasts to many other workflow orchestration engines such as conductor, airflow, Argo, Camunda, Dagster and the list goes on.
But Temporal is so much more than just a workflow orchestrator. It is a programming model that provides durable execution.  

The below code is the hello world sample from our [Java samples repository](https://github.com/temporalio/samples-java). As you can see workflows and activities are just classes. Those classes are then registered to the worker and that worker will execute in your environment, long-polling the Temporal service for workflow or activity tasks. Once the Temporal service dispatches a task it is pulled by a worker and executed. Pretty simple right?

```java
public class HelloActivity {
  static final String TASK_QUEUE = "HelloActivityTaskQueue";
  static final String WORKFLOW_ID = "HelloActivityWorkflow";

  @WorkflowInterface
  public interface GreetingWorkflow {
    @WorkflowMethod
    String getGreeting(String name);
  }

  @ActivityInterface
  public interface GreetingActivities {

    @ActivityMethod(name = "greet")
    String composeGreeting(String greeting, String name);
  }

  public static class GreetingWorkflowImpl implements GreetingWorkflow {
    private final GreetingActivities activities =
        Workflow.newActivityStub(
            GreetingActivities.class,
            ActivityOptions.newBuilder().setStartToCloseTimeout(Duration.ofSeconds(2)).build());

    @Override
    public String getGreeting(String name) {
      return activities.composeGreeting("Hello", name);
    }
  }

  static class GreetingActivitiesImpl implements GreetingActivities {
    private static final Logger log = LoggerFactory.getLogger(GreetingActivitiesImpl.class);

    @Override
    public String composeGreeting(String greeting, String name) {
      log.info("Composing greeting...");
      return greeting + " " + name + "!";
    }
  }

  public static void main(String[] args) {
    WorkflowServiceStubs service = WorkflowServiceStubs.newLocalServiceStubs();

    WorkflowClient client = WorkflowClient.newInstance(service);
    WorkerFactory factory = WorkerFactory.newInstance(client);
    Worker worker = factory.newWorker(TASK_QUEUE);

    worker.registerWorkflowImplementationTypes(GreetingWorkflowImpl.class);

    worker.registerActivitiesImplementations(new GreetingActivitiesImpl());

    factory.start();

    GreetingWorkflow workflow =
        client.newWorkflowStub(
            GreetingWorkflow.class,
            WorkflowOptions.newBuilder()
                .setWorkflowId(WORKFLOW_ID)
                .setTaskQueue(TASK_QUEUE)
                .build());

    String greeting = workflow.getGreeting("World");

    System.out.println(greeting);
    System.exit(0);
  }
}
```

## Durable Execution
Refers to Temporal's capability of maintaining the state and progress of Workflow Executions through failures, crashes, or outages using an Event History. This ensures that Workflow Executions can resume from the last recorded event, preventing loss of progress.
Lets look at the following order process. Each of the services have their own queue, state machines, retry, scheduler and timer logic. 

![Order Process](/assets/2024-05-15/order_process.png)

While each service is decoupled and blast radius, on the surface appears to be contained, it's death by a thousand cuts.

![Order Process w/Failures](/assets/2024-05-15/order_process_failures.png)

Using Temporal, we can ensure code execution is durable. The need for queues is reduced dramatically as Temporal is now orchestrating our services. Retries, schedules and timer logic no longer needs to be maintained separately within each service but rather is built-in to Temporal. Finally the business logic is maintained within workflow instead of spread across the various services. Put it all together and you get much better reliability, simplicity and as a result increased developer velocity.

![Order Process w/Temporal](/assets/2024-05-15/temporal_order_process.png)

## Determinism
Determinism in Temporal workflows refers to the principle that a workflow, given the same inputs and the same sequence of events (including external events and messages), will always produce the same output and reach the same state. This predictable behavior is crucial to Temporal's fault tolerance and recovery capabilities.
When a Temporal workflow executes, all state transitions and decisions are recorded in an Event History. If a workflow fails or is restarted, Temporal uses this Event History to "replay" the workflow from the beginning. Due to the deterministic nature of workflows, replaying the event history up to the point of failure will restore the exact workflow state at that moment, allowing the workflow to proceed as if no failure had occurred.

Workflows must be deterministic and handling non-deterministic operations is critical. In Temporal, non-deterministic operations can be handled in a side-effect (if the operation can't fail) or an activity (if the operation could fail). It is important to note that determinism only comes into play when a workflow is replayed. A replay can occur due to a failure, such as worker crashing or even the workflow being evicted from a worker's cache.


## Long Running (Entity) Workflows
Time is one of the biggest constraints we are faced with in all aspects of life. Everything has a natural cycle that is defined by either a pre-determined or non-determined amount of time. 
Temporal removes this constraint entirely for applications, as Temporal workflows only consume resources while they are actually executing operations, not waiting for for an operation to occur.

Temporal workflows are in fact "immortal", unlocking new ways to design applications. For example, imagine if you were to design a global system for orchestrating Employee Access and Permissions. An employee has a start date and an end date, however that time is not pre-determined. An employee, may for example, switch jobs or roles during their employment and as such, require new access permissions. It is a simple problem but quite challenging to solve correctly. 

What if however, the employee could be a workflow and the process easily codified? An employee Workflow could run as long as the employee remains employed in the organization. If the role changes a signal could be sent to workflow so the workflow performs appropriate CRUD operations.
Think about how much simpler onboarding or offboarding would be? Think of how easy it would be to maintain state of the employee and ensure correctness?
Entity workflows are long-running workflows, that expose state machine and represent the lifecycle of an entity. 

## Interactive Workflows
Interactive workflows are a powerful feature of Temporal that allow workflows to interact in real-time with external systems or users. This capability means that a workflow can pause its execution to wait for an external event or input before continuing. This could involve waiting for a manual approval, receiving data from a user, or integrating with another system for information necessary to proceed.

Temporal supports various types of interactions within workflows, including signals, update and queries:

### Signals 
Workflows can be designed to await signals from external sources. These signals can convey data or instructions, influencing the workflow's execution path or updating its state. For example, a workflow handling order processing might pause until it receives a signal indicating that payment has been confirmed.
Another example is pause/resume behavior. If we have a downstream failure, it might not make sense to retry and rather simply pause workflows. This could be done using a signal to pause them and another signal to resume.

### Update
Update is an enhanced form of signals. Unlike signals, which are mostly used for notifying workflows of external events, updates can trigger state changes within the workflow and are capable of returning data immediately. Update facilitates complex use cases where workflows need to adjust their behavior based on external input and provide immediate feedback such as real-time decision making applications.
Additionally Updates also alow for validation on inputs so you can from workflow reject an update where a signal is fire and forget.

### Queries 
External systems or users can query the state of a running workflow, allowing for external visibility into its progress or state. This is useful for monitoring purposes or to make decisions based on the current state of the workflow.
For example, back to our Employee Access and Permissions, the employee role state could be maintained in Temporal workflow. If an employee role changed and that change was in progress, the progress could be exposed via a query to other external systems.

## Durable Timers
A durable timer in Temporal refers to a timer that is fully managed by Temporal, ensuring reliability and durability across workflow executions. This means that even if your worker or Temporal cluster goes down when the timer is supposed to fire, the operation will not be lost. As soon as the worker and cluster are back, the workflow execution that awaits the timer will proceed. This is in stark contrast to a typical in-process timer or a timer used in traditional applications that may rely on the process's runtime and can be lost if the process crashes or restarts before the timer fires. For example, a programming language sleep operation.

The difference between a durable timer and a "normal" timer (e.g. those used in standalone applications without Temporal) lies in persistence and reliability. A normal timer's lifecycle is tied to the process it runs in and it does not survive process restarts or failures. On the other hand, Temporal's durable timer is orchestrated by the Temporal system, designed to survive failures, restarts, and ensure the timer fires as expected regardless of the state of the worker process or Temporal service. This reliability is crucial for long-running workflows and tasks that depend on accurate timing for delays, scheduling future actions, or orchestrating timeouts.

## Workflow History
Temporal Workflow History is a log of events that records every state transition and decision taken during the execution of a workflow. Each event in the history represents a specific action or change in the workflow, such as the start of the workflow, a task being scheduled, a task completing, or the workflow itself completing. This detailed record is crucial for Temporal's fault tolerance and recovery capabilities, allowing the system to resume any interrupted workflow from the last known state accurately. The Event History ensures the durability of workflow executions, enabling them to recover from failures and ensuring they can be executed exactly once to completion, regardless of the duration or the occurrence of errors during execution. This log not only aids in error handling and recovery but also serves as an audit trail for debugging and analysis purposes.

Temporal provides various views for observing a workflow's history: timeline, compact or the entire history.

### Timeline
Shows Temporal primitives across timeline. Useful for understanding how long various primitives ran.
![Timeline view](/assets/2024-05-15/timeline.png)

### Compact
Show Temporal primitives and allows for expanding them, to see all the events related to a given primitive. For example each activity has a ActivityTask scheduled, started and completed events.
![Compact view](/assets/2024-05-15/compact.png)

### History
Shows complete Temporal event history. Can be sorted in ascending or descending order. Shows all primitives, markers, activity and workflow tasks.
![History view](/assets/2024-05-15/history.png)

## Replay
Replay in Temporal is a fundamental part of ensuring durable and reliable workflow executions. When a workflow executes, each decision and state transition it makes is recorded in an Event History. If a workflow fails or is restarted, Temporal uses this Event History to replay the workflow execution from the beginning. Replay  ensures that the workflow execution is restored to its exact state at the moment of failure, allowing it to continue as if no failure had occurred. Replay is crucial for Temporal's ability to recover from failures without losing state or progress. It ensures that workflows are not only durable, but can be confidently used for long-running and mission-critical applications.

Replay is built-in to the Temporal SDK. Unless looking carefully, users would not even notice replay occurring.

## Summary
In this article we discussed the various concepts of Temporal. Temporal is a workflow orchestration system of record built on an event history. Temporal ensures workflows progress to completion regardless of failures encountered along the way. Ensuring execution progresses to completion is durable execution. Replay is the "immortality" feature that allows workflows to recover from any failure and pickup where they left off, as if nothing ever happened. Temporal is code and provides a powerful programming model, delivered through the SDK, in the programming language of choice. Using Temporal, developers can build fault-tolerant applications, that scale, at a much faster velocity than would otherwise be possible.

(c) 2024 Keith Tenzer




