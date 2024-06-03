--- 
layout: single
title:  "Temporal Graceful Worker Shutdown"
categories:
- Temporal
tags:
- Worker
- Graceful Shutdown
- Reliability
- Durability
---

![Temporal](/assets/2022-08-15/logo-temporal-with-copy.svg)

## Overview
Temporal ensures business processes are reliable and durable. It is often discussed how Temporal ensures reliability when things fail or unexpected events occur. However what about planned events? This article will focus on graceful worker shutdown and how to handle planned events in a graceful manner.

## Non-Graceful Worker Shutdown
In order to understand graceful shutdown, let us first discuss non-graceful termination of a Temporal worker. If you were to just terminate (kill -9) your Temporal worker, any workflow or activity tasks running on that worker would be timed out as the worker could no longer complete any tasks.

In the event of workflow tasks, they would be rescheduled by Temporal and picked up by another worker. The new worker would perform workflow replay to recover workflow state and resume execution from point where it left off.

In the event of activity tasks, they would timeout and be rescheduled on another worker according to activity retry policy.

## Graceful Worker Shutdown
The Temporal SDK provides hooks which allow a Temporal worker to catch termination events and gracefully shutdown. In such cases, a Temporal worker will block the shutdown and continue processing existing tasks for a configured amount of time. In addition it will stop pulling new tasks.

### Handling Activity Tasks Gracefully
The shutdown event trigger varies depending on programming language. In Java, using springboot, this is done by implementing an ApplicationListener which will trigger when a termination event or SIGTERM is received.

```java
@EventListener
@Order(Ordered.HIGHEST_PRECEDENCE)
public void handleContextClosedEventFirst(ContextClosedEvent event) {
    log.info("**** ContextClosedEvent received. ORDER(HIGHEST)");
    try {
        WorkerFactory workerFactory = event.getApplicationContext().getBean(WorkerFactory.class);
        log.info("Gracefully shutting down WorkerFactory...");

        // Initiate orderly shutdown
        workerFactory.shutdown();

        // Block until all tasks are completed (or 20 second timeout)
        workerFactory.awaitTermination(20, TimeUnit.SECONDS);

        log.info("Graceful shutdown complete!");
    } catch (BeansException e) {
        log.error("Failed to get WorkerFactory bean", e);
    }
}
```

The Temporal SDK provides the awaitTermination option which will block the shutdown for the period of time, ensuring any currently running activity tasks can complete. This means in the event of a planned worker shutdown, no activities would timeout. This not only saves time but also potentially money, in case of Temporal cloud, since retries count as an action.

### Handling Workflow Tasks Gracefully
Workflow tasks are a bit more nuanced. Workflow tasks have a timeout of 10 seconds with unlimited retries. Workflow tasks also do not count as an activity from a Temporal cloud billing perspective. As such, for most use cases, there is no need to ensure these workflow tasks are handled gracefully. However, if you have very short lived workflows (seconds) and workflow completion latency is critical, then you may want to ensure workflow tasks do not timeout when a worker is shutdown.

The Java SDK provides two options for controlling workflow tasks during graceful shutdown.  

#### RPC Long Poll Timeout
First we can control the worker RPC long poll timeout. This is how long a worker will poll the Temporal service for work (tasks) before timing out and starting a new poll.

```java
@Bean
public TemporalOptionsCustomizer<WorkflowServiceStubsOptions.Builder>
customServiceStubsOptions() {
    return new TemporalOptionsCustomizer<WorkflowServiceStubsOptions.Builder>() {
        @Nonnull
        @Override
        public WorkflowServiceStubsOptions.Builder customize(
                @Nonnull WorkflowServiceStubsOptions.Builder optionsBuilder) {
            log.info("Customizing WorkflowServiceStubsOptions");
            optionsBuilder.setRpcLongPollTimeout(Duration.ofSeconds(10));
            return optionsBuilder;
        }
    };
}
```

#### Sticky Task Queue Drain Timeout
The second option is the sticky task queue drain timeout. This will instruct worker how long to continue processing workflow tasks in event of shutdown.

```java
@Bean
public WorkerOptionsCustomizer customWorkerOptions() {
    return new WorkerOptionsCustomizer() {
        @Nonnull
        @Override
        public WorkerOptions.Builder customize(
                @Nonnull WorkerOptions.Builder optionsBuilder,
                @Nonnull String workerName,
                @Nonnull String taskQueue) {
            log.info("Customizing WorkerOptions for workerName={}, taskQueue={}", workerName, taskQueue);
            optionsBuilder.setStickyTaskQueueDrainTimeout(Duration.ofSeconds(15));
            return optionsBuilder;
        }
    };
}
```

The sticky task queue drain timeout should be shorter than the RPC long poll timeout. The recommendation is 5 seconds. 

In this example, we have chosen to give graceful shutdown (awaitTermination) 20 seconds to complete. As such, the RPC long poll timeout (setRpcLongPollTimeout) should be 15 seconds which should be shorter than the awaitTermination period. Finally, we have given 10 seconds to complete any workflow tasks (setStickyTaskQueueDrainTimeout). The setStickyTaskQueueDrainTimeout should be shorter than setRpcLongPollTimeout.

## Demonstrating Graceful Shutdown
Together with my colleague Peter Sullivan we created a [Graceful Worker Shutdown Example](https://github.com/pvsone/temporal-java-shutdown) that show cases Temporal graceful worker shutdown. In our example, we have set max retries for activities to 1. This means if a single activity times out due to shutdown or anything else the corresponding workflow would fail.

```java
private final RetryOptions retryOptions = RetryOptions.newBuilder()
        .setMaximumAttempts(1)
        .build();
```

Below we can observe the running sample application while one of the two workers is terminated using CTRL-C. We observe that the SIGTERM from our CTRL-C is caught and a graceful worker shutdown occurs. All workflows complete in 2 seconds and there are no failures. 
If a single workflow task timed out, we would see workflow completion of over 10 seconds, as the workflow task timeout is 10 seconds. In addition, if a single activity timed out we would observe a workflow failure as we have max retries set to 1. As neither happens, we can be sure that our worker shutdown was in fact, graceful.

![Graceful Shutdown](/assets/2024-05-24/temporal_graceful_worker_shutdown.gif)

## Summary
In this article we discussed Temporal graceful worker shutdown. We also explained how Temporal worker shutdown can be utilized to ensure workflow and activity tasks complete gracefully when a worker shutdown is initiated. Finally, an example was provided showing the behavior for Temporal graceful worker shutdown.

(c) 2024 Keith Tenzer




