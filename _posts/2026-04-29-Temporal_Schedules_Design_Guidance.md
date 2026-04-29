--- 
layout: single
title:  "Temporal Schedules: Design Guidance"
categories:
- Temporal
tags:
- Temporal
- Schedules
- Orchestration
- Durable Execution
---

![Temporal](/assets/2022-08-15/logo-temporal-with-copy.svg)

## Overview

Temporal Schedules are the modern, recommended way to run recurring or time-triggered workflows on Temporal. They replace older approaches like Temporal Cron Jobs, Kubernetes CronJobs, systemd timers, and Celery, while adding durability, observability, and lifecycle controls that traditional schedulers lack. This post walks through the practices we recommend for designing with Schedules, when to reach for a Schedule versus a Timer or a long-running workflow, and the design considerations (scale, state continuity, overlapping activities) that come up most often in real systems.

## General Practices for Schedules

* **Prefer Schedules over Cron** for most use cases. Schedules are designed to be more flexible and easier to operate, including support for pausing and updating.
* **Choose Schedule vs. Timer intentionally.** The key distinction is whether the delay is relative or calendar-based.
  * Use a **timer** when the delay is relative (for example, "wait 2 days") and the delay happens inside a workflow execution. Timers used in a loop within a single workflow can be cheaper than running many Schedules and make sharing state across iterations easier.
  * Use a **Schedule** when the delay applies to the entire workflow execution, is calendar-based (for example, "3 PM Wednesdays"), or is recurring.
* **Use overlap policies deliberately.** Overlap policies control what happens when a scheduled run is due to start while a previous run is still in progress. Common options include `Skip` (skip the new run), `BufferOne` (queue one pending run), `BufferAll` (queue all pending runs), `CancelOther` (cancel the in-flight run), `TerminateOther` (terminate the in-flight run), and `AllowAll` (let runs overlap freely). If you allow overlaps, make sure you understand what "last completion result" means when multiple runs can be in flight. Note that chaining state across runs via last completion result is nuanced and not commonly used in practice.
* **Add jitter** to avoid thundering herds when many schedules fire at the same time.
* **Treat schedules as declarative config** that is managed as code:
  * Define schedules alongside workflow definitions.
  * Create or update schedules through a controlled mechanism, such as CI/CD or a dedicated "schedule management" workflow, rather than manually editing in the UI while also auto-updating them.
* **Use stable, unique Schedule IDs** to avoid conflicts and make updates safe.
* **Avoid the "run limit = 1 schedule" pattern for delayed start.** For a one-time delayed workflow start, use the `StartWorkflow` start-delay option instead. It is cleaner than embedding an initial timer in your application code, and cheaper and easier to scale than a single-run Schedule.
* **Know your scaling limits and action rates.** If you will have many schedules, consider the total actions-per-second load and design accordingly.

## Scheduling Workflows

### Use Temporal Schedules Over Traditional Cron

Temporal's Schedules feature is the recommended approach for recurring workflow execution. It supersedes both Temporal Cron Jobs (now legacy) and external schedulers such as Kubernetes CronJobs, systemd timers, and Celery. \[[Schedules vs Cron](https://temporal.io/blog/temporal-schedules-reliable-scalable-and-more-flexible-than-cron-jobs)\]

Schedules offer capabilities that traditional cron cannot:

* **Lifecycle management.** Start, pause, resume, backfill, update, and delete schedules independently of Workflow Executions.
* **Overlap policies.** Control what happens when a scheduled run is due to start while a previous run is still in progress. Options include `Skip`, `BufferOne`, `BufferAll`, `CancelOther`, `TerminateOther`, and `AllowAll` (see *General Practices* above for definitions).
* **Visibility.** Every schedule's history and runs are accessible through the Temporal UI and CLI.
* **Pause-on-failure.** Schedules can automatically pause when failures occur.
* **No external dependencies.** There is no additional infrastructure to maintain.

### When to Embed Logic in Temporal vs. External Systems

#### Use Temporal Schedules When

* You need **durability and reliability**. If a scheduled run is missed (for example, due to an extended outage when no workers are available to pick up the task), Temporal can backfill it. Note that a transient single-worker crash does not require backfill: other workers will pick up the task via normal task timeout and retry.
* You want **observability**. Every run is visible in the Temporal UI.
* Your jobs have **multiple steps** or require retries, timeouts, or compensations.
* You want to **manage schedules as code** through CI/CD, for example by upserting schedules on deploy.
* You need to **dynamically update timing per entity**, such as per-user polling intervals. In this case, a long-running workflow with sleep or sleepOrWakeOnSignal may be preferable to a Schedule, especially at very large scale. 

#### Use a Long-Running Workflow Instead of a Schedule When

* The interval is **dynamic and changes frequently** per entity.
* You need **fine-grained per-entity control**, such as the ability to pause, resume, or cancel individually.
* You need to **maintain state shared across all executions** that may need to be queried or modified from outside via signals, updates, or queries. `LastCompletionResult` only exposes prior state at the start of the next run; it cannot be inspected or mutated externally.
* You are managing **very large numbers of entities** (100k+). Be aware of provisioning requirements for both approaches. 

#### Use Kubernetes CronJobs or Cloud Run When

* You have **simple, stateless, short-lived jobs** with no need for retries, observability, or multi-step logic.
* You have **existing infrastructure** already managing these jobs and no compelling reason to migrate.

That said, the Temporal documentation recommends migrating existing Kubernetes CronJobs and systemd timers to Temporal Schedules precisely because Temporal adds the durability, visibility, and control that those systems lack. \[[Migrating K8s CronJobs](https://temporal.io/blog/how-to-convert-your-job-scheduling-system-to-temporal-schedules#example-migration-from-kubernetes-cronjobs-to-temporal)\]

## Migrating Existing Cron Jobs

If you have existing cron-based jobs, wrapping them is straightforward. Wrap your existing script or job in a single-Activity Workflow, then create a Temporal Schedule to trigger it. Ensure the activity is idempotent, or disable the retry policy, to avoid duplicate side effects. \[[Migration guide](https://temporal.io/blog/how-to-convert-your-job-scheduling-system-to-temporal-schedules)\]

## Design Considerations

### Scale

* Temporal can handle millions of Schedules without issue, but your cluster or namespace needs to be provisioned to handle the burst rate if many fire simultaneously.
* At very large scale (100k+ entities), consider spreading timer firings and provisioning the Worker service role appropriately. 

### Workflow State

Each Schedule action starts a separate, independent Workflow Execution. There is a specific mechanism designed to handle state continuity between them: Last Completion Result.

### Passing State Between Scheduled Runs

A Workflow started by a Schedule can retrieve the result returned by the most recent successful run using `LastCompletionResult`. This lets you effectively chain state across independent executions. \[[Last completion result](https://docs.temporal.io/schedule#last-completion-result)\]

### Key Things to Know About Last Completion Result

* **Failures and timeouts do not affect it.** Only successful completions update the last completion result, so a failed run will not overwrite the last good state. 
* **Size limit.** Results up to approximately 2 MiB are supported, but results within 1 KiB of that limit will not be passed to the next execution.
* **Continue-As-New caveat.** If a scheduled Workflow uses Continue-As-New, `LastCompletionResult` will not be accessible in the new iteration.
* **Overlap policy semantics.** For non-overlapping policies, "most recent successful run" is straightforward. For `AllowAll`, it refers to whichever run completed most recently at the time the new run started. Note that start time is irrelevant: a run that started earlier but takes a long time can complete *after* a later-started run, in which case the earlier-started-but-later-completed run is the "most recent" one whose result is exposed. This is a common source of confusion.

### When This Is Not Enough

If you need richer or more complex state shared across runs, the alternative is to use a single long-running Workflow (with Continue-As-New to manage history size) rather than Schedules. This keeps all state within one execution context. The tradeoffs are:

* You give up Schedule-specific features such as pause, resume, backfill, and overlap policies.
* Calendar-based scheduling is not directly available. A long-running workflow uses relative timers (for example, `sleep` / `workflow.sleep`), so calendar-style schedules like "every Wednesday at 3 PM" must be computed manually as a duration to the next firing time.

So the "state spread across runs" issue is real but well-addressed by `LastCompletionResult` for most use cases, particularly when your overlap policy is non-overlapping. The main exception is when you need to modify or query that state from outside (via signals, updates, or queries): `LastCompletionResult` is read-only and only available at the start of the next run, so a single long-running workflow is the better fit.

### Overlapping Activities

While overlapping activities are possible without Schedules, they become more likely when using Schedules due to factors such as worker timeouts. The same activity may run twice. As such, it is important to ensure activities are idempotent.

The concern is not limited to the same activity running concurrently. Different activities can also run concurrently when it is not expected, causing potential race conditions. There are no hard guarantees that when an activity starts, any previous attempts have completed.

These situations apply to non-scheduled workflows as well, but Schedules make them more likely.

## Summary

Temporal Schedules are the right default for almost any recurring or calendar-driven workflow. The key takeaways from this post are:

* **Default to Schedules over cron-style alternatives.** They give you durability, observability, lifecycle controls, and pause/resume/backfill out of the box, with no extra infrastructure to maintain.
* **Pick the right primitive for the job.** Use a timer for relative delays inside a workflow, a Schedule for calendar-based or recurring runs, the `StartWorkflow` start-delay for one-shot delayed starts, and a long-running workflow when you need dynamic per-entity intervals or shared state that is read or written from outside.
* **Be deliberate about overlap.** Choose your overlap policy intentionally, prefer non-overlapping policies when you can, and remember that `AllowAll` defines "most recent" by completion time, not start time.
* **Treat Schedules as code.** Manage them through CI/CD, use stable Schedule IDs, and add jitter to avoid thundering herds at scale.
* **Design activities to be idempotent.** Schedules make overlapping activity executions more likely, and Temporal does not guarantee that prior attempts have finished before a new one begins.
* **Plan for scale.** Temporal can handle very large numbers of Schedules, but your cluster, namespace, and Worker fleet need to be provisioned for the simultaneous-fire burst rate.

Used together, these practices give you a scheduling layer that is both reliable in the face of failures and easy to evolve as your system grows.