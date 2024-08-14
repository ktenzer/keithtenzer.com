--- 
layout: single
title:  "Teaching Old Workflows, New Tricks with Interceptors"
categories:
- Temporal
tags:
- Temporal
- Durable Execution
- Replay
- Determinism
- Workflow Orchestration
- Workflow as Code
- Interceptors
---

![Temporal](/assets/2022-08-15/logo-temporal-with-copy.svg)
## Overview
The old saying "You can't teach an old dog, new tricks" is true. It's hard to change behavior, the older one gets. With code, it is also similar. It is easy to do a v1 but as complexity and features grow, changing behavior becomes more challenging.

Temporal provides Interceptors, which allow us to change behavior of SDK calls. They can be likened to middleware in other frameworks, as they enable you to inject custom logic, such as logging, tracing, or enforcing security, into the Workflow and Activity execution lifecycle.

This blog could not have been written without the wonderful samples provided by my colleague Tihomir Surdilovic.

## Types of Interceptors
There are four types of Interceptors in Temporal: WorkflowClientInterceptor, WorkflowInboundCallsInterceptor, WorkflowOutboundCallsInterceptor and ActivityInboundCallsInterceptor.

**WorkflowClientInterceptor** - Intercepts Workflow client outbound calls. Specifically, calls from client to Workflow, such as, executing, signaling, updating and even querying Workflows.

**WorkflowInboundCallsInterceptor** - Intercepts Workflow inbound calls. These are calls coming to the Workflow, for example, Workflow execution, signals, update and queries

**WorkflowOutboundCallsInterceptor** - Intercepts Workflow outbound calls. These calls are coming from Workflow to Temporal service through API. As example, scheduling activities and starting timers.

**ActivityInboundCallsInterceptor** - Intercepts inbound calls to a Workflow Activity, for example, Activity execution. 

## WorkflowClientInterceptor
The WorkflowClientInterceptor intercepts all client outbound calls. It can be used for special handling of client errors, especially logging them. For example, logging all client requests a certain way, across all clients.

Another example from our Java samples repository is a [Count Interceptor](https://github.com/temporalio/samples-java/blob/main/core/src/main/java/io/temporal/samples/countinterceptor/SimpleClientCallsInterceptor.java). Here the WorkflowClientInterceptor is used to count Workflows, Signals, Queries and even Workflow Results called by the client.

```java
public class SimpleClientCallsInterceptor extends WorkflowClientCallsInterceptorBase {
  private ClientCounter clientCounter;

  public SimpleClientCallsInterceptor(
      WorkflowClientCallsInterceptor next, ClientCounter clientCounter) {
    super(next);
    this.clientCounter = clientCounter;
  }

  @Override
  public WorkflowStartOutput start(WorkflowStartInput input) {
    clientCounter.addStartInvocation(input.getWorkflowId());
    return super.start(input);
  }

  @Override
  public WorkflowSignalOutput signal(WorkflowSignalInput input) {
    clientCounter.addSignalInvocation(input.getWorkflowExecution().getWorkflowId());
    return super.signal(input);
  }

  @Override
  public <R> GetResultOutput<R> getResult(GetResultInput<R> input) throws TimeoutException {
    clientCounter.addGetResultInvocation(input.getWorkflowExecution().getWorkflowId());
    return super.getResult(input);
  }

  @Override
  public <R> QueryOutput<R> query(QueryInput<R> input) {
    clientCounter.addQueryInvocation(input.getWorkflowExecution().getWorkflowId());
    return super.query(input);
 ```

## WorkflowInboundCallsInterceptor
The WorkflowInboundCallsInterceptor can be used for use cases such as authorization, logging and validation at the beginning of a Workflow execution, or during, when a Workflow receives signals or even queries.

In our [Count Interceptor](https://github.com/temporalio/samples-java/blob/main/core/src/main/java/io/temporal/samples/countinterceptor/SimpleClientCallsInterceptor.java) below, the number of Workflows executed, in addition to signals or queries received is being counted and logged.

```java
public class SimpleCountWorkflowInboundCallsInterceptor
    extends WorkflowInboundCallsInterceptorBase {

  private WorkflowInfo workflowInfo;

  public SimpleCountWorkflowInboundCallsInterceptor(WorkflowInboundCallsInterceptor next) {
    super(next);
  }

  @Override
  public void init(WorkflowOutboundCallsInterceptor outboundCalls) {
    this.workflowInfo = Workflow.getInfo();
    super.init(new SimpleCountWorkflowOutboundCallsInterceptor(outboundCalls));
  }

  @Override
  public WorkflowOutput execute(WorkflowInput input) {
    WorkerCounter.add(this.workflowInfo.getWorkflowId(), WorkerCounter.NUM_OF_WORKFLOW_EXECUTIONS);
    return super.execute(input);
  }

  @Override
  public void handleSignal(SignalInput input) {
    WorkerCounter.add(this.workflowInfo.getWorkflowId(), WorkerCounter.NUM_OF_SIGNALS);
    super.handleSignal(input);
  }

  @Override
  public QueryOutput handleQuery(QueryInput input) {
    WorkerCounter.add(this.workflowInfo.getWorkflowId(), WorkerCounter.NUM_OF_QUERIES);
    return super.handleQuery(input);
  }
}
```

## WorkflowOutboundCallsInterceptor
The WorkflowOutboundCallsInterceptor can be used to intercept calls from Workflow, before they got back to the Temporal service. A good use case is for changing behavior of how Activity failures is handled. 

In the [example](https://github.com/temporalio/temporal-pause-resume-compensate/tree/main) below, a pause/resume feature is added to the Workflow, upon Activity failure, using this Interceptor. If a downstream goes down or an Activity fails say X times, it may be desired to pause the Workflow, instead of continuing to retry an Activity, which will fail. Here when the Activity fails, a search attribute is set indicating the Workflow is paused. The Activity is completed asynchronously and the Workflow is paused or blocked, until a Signal is received to resume execution, which will retry the failed Activity.

```java
public class PauseResumeWorkflowOutboundCallsInterceptor extends WorkflowOutboundCallsInterceptorBase {

    private final SearchAttributeKey<Boolean> PAUSE_SA =
            SearchAttributeKey.forBoolean("pause");

    private enum Action {
        RETRY,
        FAIL
    }

    private class ActivityRetryState<R> {
        private final ActivityInput<R> input;
        private final CompletablePromise<R> asyncResult = Workflow.newPromise();
        private CompletablePromise<Action> action;
        private Exception lastFailure;
        private int attempt;

        private ActivityRetryState(ActivityInput<R> input) {
            this.input = input;
        }

        ActivityOutput<R> execute() {
            return executeWithAsyncRetry();
        }

        private ActivityOutput<R> executeWithAsyncRetry() {
            attempt++;
            lastFailure = null;
            action = null;
            ActivityOutput<R> result =
                    PauseResumeWorkflowOutboundCallsInterceptor.super.executeActivity(input);
            result
                    .getResult()
                    .handle(
                            (r, failure) -> {
                                // No failure complete
                                if (failure == null) {
                                    Workflow.upsertTypedSearchAttributes(
                                            PAUSE_SA.valueUnset(), PAUSE_SA.valueSet(false));
                                    pendingActivities.remove(this);
                                    asyncResult.complete(r);
                                    return null;
                                }
                                // Asynchronously executes requested action when signal is received.
                                Workflow.upsertTypedSearchAttributes(
                                        PAUSE_SA.valueUnset(), PAUSE_SA.valueSet(true));
                                lastFailure = failure;
                                action = Workflow.newPromise();
                                return action.thenApply(
                                        a -> {
                                            if (a == Action.FAIL) {
                                                Workflow.upsertTypedSearchAttributes(
                                                        PAUSE_SA.valueUnset(), PAUSE_SA.valueSet(false));
                                                asyncResult.completeExceptionally(failure);
                                            } else {
                                                // Retries recursively.
                                                executeWithAsyncRetry();
                                            }
                                            return null;
                                        });
                            });
            return new ActivityOutput<>(result.getActivityId(), asyncResult);
        }

        public void retry() {
            if (action == null) {
                return;
            }
            action.complete(Action.RETRY);
        }

        public void fail() {
            if (action == null) {
                return;
            }
            action.complete(Action.FAIL);
        }

        public String getStatus() {
            String activityName = input.getActivityName();
            if (lastFailure == null) {
                return "Executing activity \"" + activityName + "\". Attempt=" + attempt;
            }
            if (!action.isCompleted()) {
                return "Last \""
                        + activityName
                        + "\" activity failure:\n"
                        + Throwables.getStackTraceAsString(lastFailure)
                        + "\n\nretry or fail ?";
            }
            return (action.get() == Action.RETRY ? "Going to retry" : "Going to fail")
                    + " activity \""
                    + activityName
                    + "\"";
        }
    }

    private final List<ActivityRetryState<?>> pendingActivities = new ArrayList<>();

    public PauseResumeWorkflowOutboundCallsInterceptor(WorkflowOutboundCallsInterceptor next) {
        super(next);
        Workflow.registerListener(
                new PauseResumeInterceptorListener() {
                    @Override
                    public void retry() {
                        for (ActivityRetryState<?> pending : pendingActivities) {
                            pending.retry();
                        }
                    }

                    @Override
                    public void fail() {
                        for (ActivityRetryState<?> pending : pendingActivities) {
                            pending.fail();
                        }
                    }

                    @Override
                    public String getPendingActivitiesStatus() {
                        StringBuilder result = new StringBuilder();
                        for (ActivityRetryState<?> pending : pendingActivities) {
                            if (result.length() > 0) {
                                result.append('\n');
                            }
                            result.append(pending.getStatus());
                        }
                        return result.toString();
                    }
                });
    }

    @Override
    public <R> ActivityOutput<R> executeActivity(ActivityInput<R> input) {
        ActivityRetryState<R> retryState = new ActivityRetryState<R>(input);
        pendingActivities.add(retryState);
        return retryState.execute();
    }
}
```

## ActivityInboundCallsInterceptor
The use cases for the ActivityInboundCallsInterceptor are logging, monitoring or even collecting performance metrics, such as Activity execution time. In the below example, the Activity inbound call is intercepted, so that the total Activity execution time can be logged.

```java
public class SimpleCountActivityInboundCallsInterceptor
    extends ActivityInboundCallsInterceptorBase {

  private ActivityExecutionContext activityExecutionContext;

  public SimpleCountActivityInboundCallsInterceptor(ActivityInboundCallsInterceptor next) {
    super(next);
  }

  @Override
  public void init(ActivityExecutionContext context) {
    this.activityExecutionContext = context;
    super.init(context);
  }

  @Override
  public ActivityOutput execute(ActivityInput input) {
        long startTime = System.currentTimeMillis();
        try {
            return super.execute(input);
        } finally {
            long endTime = System.currentTimeMillis();
            System.out.println("Activity " + input.getMetadata().getActivityType() + " took " + (endTime - startTime) + " milliseconds");
        }
    }
```

## Excluding Workflow and Activity Types
By default, Interceptors are global across all Workflow and Activity types registered to a worker. Interceptors are configured on the worker using the WorkflowClientOptions stub.

```java
WorkflowClient client =
    WorkflowClient.newInstance(
        service, WorkflowClientOptions.newBuilder().setInterceptors(clientInterceptor).build());
```

However, it is also possible to exclude certain Workflow and Activity types from Interceptors. This makes it possible to have different implementations of the same Interceptor and then apply those Interceptors to different Workflow or Activity types. 

The [Exclude From Interceptor](https://github.com/temporalio/samples-java/tree/main/core/src/main/java/io/temporal/samples/excludefrominterceptor) sample demonstrates how to handle exclusion.

First we need to handle excluding Workflow and Activity types.
```java
public class MyWorkerInterceptor implements WorkerInterceptor {
  private List<String> excludeWorkflowTypes = new ArrayList<>();
  private List<String> excludeActivityTypes = new ArrayList<>();

  public MyWorkerInterceptor() {}

  public MyWorkerInterceptor(List<String> excludeWorkflowTypes) {
    this.excludeWorkflowTypes = excludeWorkflowTypes;
  }

  public MyWorkerInterceptor(List<String> excludeWorkflowTypes, List<String> excludeActivityTypes) {
    this.excludeWorkflowTypes = excludeWorkflowTypes;
    this.excludeActivityTypes = excludeActivityTypes;
  }

  @Override
  public WorkflowInboundCallsInterceptor interceptWorkflow(WorkflowInboundCallsInterceptor next) {
    return new MyWorkflowInboundCallsInterceptor(excludeWorkflowTypes, next);
  }

  @Override
  public ActivityInboundCallsInterceptor interceptActivity(ActivityInboundCallsInterceptor next) {
    return new MyActivityInboundCallsInterceptor(excludeActivityTypes, next);
  }
}
```

Finally when creating worker, Interceptors are set using the above class, which handles the exclusions that are passed in.
```java
WorkerFactoryOptions wfo =
    WorkerFactoryOptions.newBuilder()
        // exclude MyWorkflowTwo from interceptor
        .setWorkerInterceptors(
            new MyWorkerInterceptor(
                // exclude MyWorkflowTwo from workflow interceptors
                Arrays.asList(MyWorkflowTwo.class.getSimpleName()),
                // exclude ActivityTwo and the "ForInterceptor" activities from activity
                // interceptor
                // note with SpringBoot starter you could use bean names here, we use strings to
                // not have
                // to reflect on the activity impl class in sample
                Arrays.asList(
                    "ActivityTwo", "ForInterceptorActivityOne", "ForInterceptorActivityTwo")))
        .validateAndBuildWithDefaults();
```
## Summary 
In this article we reviewed the four types of Interceptors: WorkflowClientInterceptor, WorkflowInboundCallsInterceptor, WorkflowOutboundCallsInterceptor and ActivityInboundCallsInterceptor that can be configured within a Temporal worker. We explained some of the use cases and provided several examples. Finally, it was demonstrated how types can be excluded, providing maximum flexibility and reusability of Interceptors.

(c) 2024 Keith Tenzer




