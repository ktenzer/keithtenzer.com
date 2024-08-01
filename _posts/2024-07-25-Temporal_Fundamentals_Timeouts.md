--- 
layout: single
title:  "Temporal Fundamentals Part III: Timeouts"
categories:
- Temporal
tags:
- Temporal
- Durable Execution
- Replay
- Determinism
- Workflow Orchestration
- Workflow as Code
- Timeouts
---

![Temporal](/assets/2022-08-15/logo-temporal-with-copy.svg)
## Overview
This is a five part series focused on Temporal fundamentals. It represents, in my words, what I have learned along the way and what I would've like to know on day one.

- [Temporal Fundamentals Part I: Basics](https://keithtenzer.com/temporal/Temporal_Fundamentals_Basics)
- [Temporal Fundamentals Part II: Concepts](https://keithtenzer.com/temporal/Temporal_Fundamentals_Concepts/)
- [Temporal Fundamentals Part III: Timeouts](https://keithtenzer.com/temporal/Temporal_Fundamentals_Timeouts/)
- Temporal Fundamentals Part IV: Workflows
- Temporal Fundamentals Part V: Workers

This article will focus on understanding timeouts and their impact on Temporal workflow execution. 

## Workflow Timeouts
There are several timeouts that can be configured for Temporal workflows. The timers for these timeouts are handled by the Temporal service and as such are not visible to the workers.

### Workflow Execution Timeout
Total workflow execution timeout, including retries and any invocations of continue-as-new which generates a new run-id. The default for this timeout is unlimited. It is not recommended to use workflow execution timeout for business logic.
If for example, you need to timeout a workflow after X minutes, it would be recommended to use a timer instead. The reason is that the workflow execution timeout cannot be caught in workflow code, the execution will be terminated instead from the service and workers will not be notified, meaning cancellation or cleanup is also not possible.
Use cases where it is recommended to use workflow execution timeout is when duration of workflow retries needs to be limited, with cron to limit duration of execution or with continue-as-new and finally possibly for integration or load testing.

### Workflow Run Timeout
Timeout for a single workflow execution or run. Cannot be greater than workflow execution timeout. The default is unlimited and again it is not recommended to change or set this timeout for same reasons as with workflow execution timeout.
The use cases of when to use this timeout is also similar to that of the workflow execution timeout. The only difference being we are limiting a specific run or execution.

### Workflow Task Timeout
Timeout of a single workflow task. Workflow execution is a coordination between workers and the Temporal service to progress a workflow to completion. Workflow tasks are responsible for progressing your workflow code. When a workflow is started, a workflow task is scheduled and picked up by a worker. Workflow code is executed by a workflow task until it reaches a Temporal primitive (such as ExecuteActivity) where it returns next steps to the Temporal service for coordination.
The workflow task timeout, is the time from when a worker picks up a workflow task, until it responds with completion. The default for the workflow task timeout is 10 seconds and the maximum time is two minutes.
It is critical that any code which can fail, happens inside an Activity and not workflow code which is coordinated through workflow tasks. In addition workflow code should never block as that could result in a loop causing a stuck workflow that never progresses.
It is not recommended to change this timeout unless there is a very specific use case. 

![Workflow Task Timeouts](/assets/2024-07-25/workflow_task_timeouts.gif)

## Activity Timeouts
In Temporal, any code that can fail should be executed inside an activity. Activities have granular retry policies but also several important timeouts that affect execution. Activities have three possible states: scheduled, started and completed. 

It is required that an Activity set either Start-to-Close or Schedule-to-Close timeout.

![Activity Timeouts](/assets/2024-07-25/activity_timeouts.gif)

### Start-to-Close Timeout
This timeout is the length of time from when the activity was started to when it is completed.

This timeout should be set to just slightly longer than what we expect activity to run for.

### Schedule-to-Close Timeout
This timeout is the length of time from when the activity was scheduled to when it is completed.

As with Start-to-Close timeout, it should be set to just slightly longer than what we expect activity to run for. If there are no workers or workers are busy, activity will sit on queue and remain scheduled so if that is possibility, it should also be taken into account.

### Schedule-to-Start Timeout
This timeout is the length of time from when the activity was scheduled to when it started.

This timeout should only be used in special cases, for example re-routing activity to a different task queue, if it is stuck on the queue and in a scheduled state. Since Temporal workers can be scaled horizontally however, it is better to go with scaling approach (if possible) and simply scale up workers on task queue, as opposed to re-routing Activities to other queues.

### Heartbeat Timeout
This timeout measures the amount of time since the last heartbeat sent from an activity. This timeout is useful for long-running activities, where it is important to reschedule activity, if there is a worker issue well in advance of Start-to-Close or Schedule-to-Close, which could be set to hours or even days.

Heartbeat timeout lets Temporal service reschedule activity quickly, in the event of some worker or infrastructure failure causing activity to not be completed.

If setting heartbeat timeout, it is critical to also heartbeat from the activity, otherwise timeout is ignored. Activity should heartbeat more frequently that the StartToClose timeout.

SDK throttles heartbeats at 80 percent.

## Local Activity Timeouts
Local activities run inside of workflow tasks. They are used for short lived read/write operations, for example persisting to a database.

They do not support heartbeats and are not suited for longer running activities.

Due to retries extending workflow task, local activities can cause delayed delivery of signals and even timeout query requests. If using signals or queries with local activities it is important to consider the possible impact.

### Start-To-Close
This timeout is the length of time from when the local activity was started to when it is completed.

If local activity runs longer than the workflow task timeout, which defaults to 10 seconds, timers will be used to extend workflow task. Again it is strongly recommended to avoid going beyond the 10 seconds and extending workflow task timeout. As such, we should consider limiting retries of local activities.

### Schedule-To-Close
This timeout is the length of time from when the local activity was scheduled to when it is completed.

Same as with normal activity timeout, it should be set to just slightly longer than we expect Activity to run and as mentioned above, we want to be careful about extending workflow task timeout beyond the default 10 seconds.

## Summary
In this article we discussed Temporal timeouts. There are two categories of timeouts, those that can be configured on the workflow and those that can be configure on the activity. When configuring workflow timeouts, it is important to understand they are not visible to the worker and as such catching them and handling cancellation is not possible. When configuring activity timeouts it is important to understand the difference between activities and local activities. Appropriate configuration of timeouts, according to requirements is critical for proper function of Temporal workflows.  

(c) 2024 Keith Tenzer




