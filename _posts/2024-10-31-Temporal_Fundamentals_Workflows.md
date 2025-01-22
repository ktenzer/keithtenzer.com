--- 
layout: single
title:  "Temporal Fundamentals Part IV: Workflows"
categories:
- Temporal
tags:
- Temporal
- Durable Execution
- Replay
- Determinism
- Workflow Orchestration
- Workflow as Code
- Workflow
---

![Temporal](/assets/2022-08-15/logo-temporal-with-copy.svg)
## Overview
This is a six part series focused on Temporal fundamentals. It represents, in my words, what I have learned along the way and what I would've like to know on day one.

- [Temporal Fundamentals Part I: Basics](https://keithtenzer.com/temporal/Temporal_Fundamentals_Basics)
- [Temporal Fundamentals Part II: Concepts](https://keithtenzer.com/temporal/Temporal_Fundamentals_Concepts/)
- [Temporal Fundamentals Part III: Timeouts](https://keithtenzer.com/temporal/Temporal_Fundamentals_Timeouts/)
- [Temporal Fundamentals Part IV: Workflows](https://keithtenzer.com/temporal/Temporal_Fundamentals_Workflows/)
- [Temporal Fundamentals Part V: Workflow Patterns](https://keithtenzer.com/temporal/Temporal_Fundamentals_Workflow_Patterns/)
- [Temporal Fundamentals Part VI: Workers](https://keithtenzer.com/temporal/Temporal_Fundamentals_Workers/)

This article will focus on understanding Temporal Workflow design recommendations. It will explore general design practice and also get into specific recommendations for Temporal primitives.

![Temporal Workflow](/assets/2024-10-31/workflow.png)

## Limits
Temporal achieves its breathtaking scale, by its ability to execute billions upon billions of Workflows. However, each Workflow has its [limits](https://docs.temporal.io/cloud/limits#programming-model-level) and it is important to understand them, when doing Workflow design.

## Determinism
Workflows must be deterministic, Workflows can and will be replayed. Every Workflow replay must follow the same code path, for events that already ocurred in its execution history. 

**Recommendations**
- Use static code analyzers if available for SDK, which check for common sources of Non-Determinism.
- Workflows should be replayed using the SDK replayer, with previous version event history, to ensure new code doesn't break determinism. 
- Versioning Workflows should be done using appropriate versioning strategy.

## WorkflowIds
WorkflowIds are unique and often they represent a name or identifier, that is important to the use case. Workflows have both, a WorkflowId, and a RunId. While the WorkflowId only exists once, each run including retry or continue-as-new of workflow, creates a new RunId.

**Recommendations**
- Use WorkflowIds as idempotency keys when Workflow may be started more than once.
- Use appropriate WorkflowId [reuse](https://docs.temporal.io/workflows#workflow-id-reuse-policy) policy
- Do not rely on Workflow RunId for business logic choices as it can change during Workflow execution, due to retry or continue-as-new.

## Workflow Event History
A Workflow is comprised of a series of tasks. Workflow tasks execute Workflow code and Activity tasks execute Activity code. All of these tasks and their states (Scheduled, Started, Completed) are stored in the Workflow event history. A single Workflow event history has limits on number of events (50,000) and size (50 MB).

**Recommendations**
- Ensure Workflow limits are not reached, otherwise Workflow will be terminated. 
- Partition work across Workflows as opposed to having a single monolithic Workflow. In Temporal, you can have billions of running workflows, use them.
- Long running Workflows should use Continue-As-New within Workflow to continue with fresh event history or Child Workflows, to partition event history across many workflows.

## Workflow Timeouts
Workflow tasks have a timeout of 10 seconds by default. In addition, the SDK has a deadlock detector which is 1 second. Workflow execution and run timeouts default to infinite.

**Recommendations**
- Do not change defaults unless you have a very specific reason.
- It is very important to not block Workflow code due to these important timeouts, as that will block and potentially cause stuck Workflows. 

## Workflow Failure
Workflow code that throws a non-Temporal failure will cause Workflow task failure. By default Workflow task will be retried every 10 seconds, infinity until it succeeds.

**Recommendations**
- Properly catch and throw exceptions. Can throw Temporal Application error if it is decided to fail Workflow.
- Don't use Workflow retry policy, a Workflow should not fail due to intermittent issues.

## Workflow Primitives
### Activities
A Temporal Activity is used to call external services or APIs. Anything that can fail must be an Activity. Anything that could be non-Deterministic must also be done in an Activity.

**Recommendations**
- Activities should be idempotent. In Temporal you can have at most once Activity execution (0 or 1) or at least once Activity execution (1 or more).
- Long running activities (more than few minutes) should always Heartbeat.
- Ensure proper Activity timeout for use case (StartToClose or ScheduleToClose).
- Ensure proper failure handling, retry policy, non-retryable errors and compensation is appropriate for use case.
- Activity payloads have limit of 2 MB. if payloads, inputs/outputs of Activity could be more, consider passing by reference. Another option could be compression using Data Converter.
- For polling, if frequent perform polling within activity with iterator. If infrequent perform polling using Activity retry. You can also consider Async completion approach, which allows Activity function to return without completing it.
- Use Local Activity for very short-lived Activities, where latency is important. For example, database write. Local Activities run under a Workflow task and cannot Heartbeat.

### Child Workflows
Temporal Workflows can partition themselves and create a Child Workflow. This creates a relationship between the parent and the child. Child Workflows can also create other Child Workflows for which they become a parent. A parent and child Workflow have relationship, and the Parent-Close-Policy determines what happens to a Child Workflow if its parent completes, fails or is timed out.

**Recommendations**
- Use Child Workflows to partition into smaller chunks in order to stay within Workflow event history limits.
- Use Child Workflows when lifecycle varies between Workflows (order vs shipment), to breakup and keep Workflow manageable, potentially when multiple teams are involved in use case.
- Do not use Child Workflows to organize code, use programming language for that. Child Workflows will result in more events and actions so they are more expensive than just using activities.

### Signal
A Signal is used to update or mutate a Workflow. This is one of the ways Workflows can be interacted with externally, either from another Workflow or the Temporal client. Signals have important behavior characteristics, they are fire and forget, but do have an order guarantee.

**Recommendations**
- Don't send many Signals per second for extended periods of time. As Signals are buffered, they must be processed prior to a Continue-As-New, which can result in that not happening, and as a result the event history reaching it's limit.
- A Workflow is limited to 10,000 Signals received.
- Not recommended to have Signal call Activity, should limit scope of Signal handler to updating Workflow state and let Workflow code react to state changes.

### Update
Similar to Signals, Update is used to mutate a Workflow and allows Workflows to be interacted with externally. Where it differs from Signal is Update can return a response to the client/caller and do validation. If validation for example is rejected, the update would fail and the is error returned to the client/caller. With Signals, since they are fire and forget, you can easily flood or overwhelm a Workflow if you aren't careful.

**Recommendations**
- Updates that are rejected are not recorded in the event history.

### Query
In Temporal, any data structure maintained in the Workflow, can be exposed externally, using a Query. A Query is an asynchronous operation used to get the state of a Workflow execution. Queries work on both running and completed Workflows. A worker is required to respond to Query. If no workers are running, the Query call will fail.

**Recommendations**
- Queries should never mutate state of Workflow, and should be read-only.
- It is not recommended to continuously poll using a Query and instead use more efficient patterns.
- Queries are not recorded in the event history.

### Timer
A Timer in Temporal is a durable sleep, maintained by the Temporal service. Timers are used to delay execution within a Workflow, or make business logic decisions based on time. For example, failing a Workflow that doesn't complete in X time, or cancelling an Activity that doesn't complete in Y time.

**Recommendations**
- Never use the programming language sleep in Workflow, instead use Workflow.timer to sleep Workflow.
- When using timers for business logic decisions, if timer doesn't fire it should be properly cancelled so it is reflected in event history.
- Use timers to cancel and fail Workflows that run too long, instead of Workflow timeout.

### Continue-As-New
The Continue-As-New primitive in Temporal allows for continuing Workflow, with a fresh or new event history. This is useful for long-running Workflows to prevent reaching event history limits. Continue-As-New allows for passing Workflow state from current runId/execution to a new one. As such, it can also be used for Workflow migration, as well as, other such use cases where passing Workflow state, and continuing in a new Workflow is advantageous.

**Recommendations**
- Ensure the required state is being passed in to Continue-As-New and nothing is missing.
- Allow Temporal SDK to automatically Continue-As-New, when it thinks it should, for avoiding event history limits.

## Summary
In this article we discussed Temporal Workflow design principles. We explained and also provided recommendations for Temporal Workflow primitives used to build Temporal Workflows.

(c) 2024 Keith Tenzer




