--- 
layout: single
title:  "Temporal Schedules"
categories:
- Temporal
tags:
- Temporal
- Workflows
- Schedules
- Opensource
---

![Temporal](/assets/2022-08-15/logo-temporal-with-copy.svg)
## Overview
In this article we will introduce the new scheduler feature that ships with Temporal v1.18.0. Scheduling is a critical part of many workflow lifecycle's. From trip planning applications, to payments, infrastructure and beyond, scheduling is a big part of what we do on a daily basis. Having a first-class scheduler built into Temporal will undoubtedly unlock new possibilities.

## History
Historically, Temporal has supported scheduling via Cron. While cron enabled a workflow to be scheduled, it did not provide granular primitives or management/visibility. It really only worked for very simple scheduling needs. As a result, applications dependent on a true scheduler needed to get those capabilities from outside of Temporal (such as Quartz). 

Cron was easy, you simply passed in a cron schedule through your workflow options. However as you can see below, there is nothing you can configure or control.
```go
workflowOptions := client.StartWorkflowOptions{
	ID:        "backup_sample",
	TaskQueue: "backup-sample",
	CronSchedule: "* * * * *",        
}
```    

Below is a list of the cron drawbacks and why a new scheduler was needed. 
A cron workflow:
- Cannot be stopped without affecting the running workflow
- Cannot be terminated without affecting schedule
- Cannot be changed after it’s created
- Cannot be paused
- Cannot support overlapping runs (same workflow id)
- Has a single string specification, not structured data
- Start's in an odd state (started but not scheduled)

The cron feature will remain for now along with the new scheduler but at some point may be deprecated.

## New Scheduler Features
Now that we have explained the drawbacks of cron, lets talk about the capabilities the new scheduler.

- Provides visibility into schedules and workflow runs
- Schedules are independent entities (separate from the workflows they start)
- Schedule specification is structured and extensible
- Schedules can have state (Paused/Running)
- Can limit the number of runs
- Scheduler can backfill runs that would have happened during a specific time range
- Overlapping runs are possible
- Can configure policies, for example “buffer all runs”, “buffer one”, “cancel other”, etc
- Pause-on-failure

## Using the Scheduler
Schedules can be configured either by using the SDK (as of this writing gRPC protos only but SDK wrappers are coming soon) or tctl. Schedules can be viewed using tctl or the UI. In this example, we will demonstrate using tctl to configure a schedule and the UI to display it.

### Creating Schedule
First we will simply get a list of workflows that have run using tctl.

```bash
$ tctl wf l
  WORKFLOW TYPE | WORKFLOW ID            |                RUN ID                |  TASK QUEUE   | START TIME | EXECUTION TIME | END TIME  
  Workflow      | hello_world_workflowID | 3db1fe3f-2ed3-464b-a6dc-c117783bd262 | hello-world   | 17:01:48   | 17:01:48       | 17:01:48
```

Next we will schedule the above workflow to run every 5 minutes. In addition to the information gathered by tctl we will also need to provide a schedule id (sid).

```bash
$ tctl schedule create --cron "0/5 * * * *" --workflow_id hello_world_workflowID --taskqueue hello-world  --workflow_type Workflow --sid 123
```

### Viewing Schedule
Using the new schedule widget in the UI navigation bar we can see schedules listed by their corresponding schedule id.
![View of Schedules](/assets/2022-10-20/schedules.png)

From here we can drill into a schedule to see it's details. We can also see the result of recent runs and even schedule time for future upcoming runs.
![View of Schedule](/assets/2022-10-20/schedule_view.png)

The schedule can even be paused or resumed. 

In addition all of this is also possible with the tctl command.
```bash
$ tctl schedule
NAME:
   tctl schedule - Operate schedules

USAGE:
   tctl schedule command [command options] [arguments...]

COMMANDS:
   create    Create a new schedule
   update    Updates a schedule with a new definition (full replacement, not patch)
   toggle    Pauses or unpauses a schedule
   trigger   Triggers an immediate action
   backfill  Backfills a past time range of actions
   describe  Get schedule configuration and current state
   delete    Deletes a schedule
   list      Lists schedules
```  

## Summary
In this article we learned about the new scheduler feature available in Temporal v1.18.0. We looked at how to schedule a workflow using the tctl command and how to view schedules in the UI. Scheduling is very powerful in the context of workflows. Having first-class scheduler primitives built into Temporal will certainly unlock many new possibilities.

(c) 2022 Keith Tenzer




