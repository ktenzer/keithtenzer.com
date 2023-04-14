--- 
layout: single
title:  "Temporal Time Traveling: Replay"
categories:
- Temporal
tags:
- Temporal
- Workflows
- Replay
---

![Temporal](/assets/2022-08-15/logo-temporal-with-copy.svg)
## Overview
In this article we will discuss one of the resilience features of Temporal called replay. Temporal replay refers to the process of re-executing a workflow, or part of a workflow, from the recorded state of the workflow history. Replay enables reconstruction of workflow history for the purpose of state analysis, debugging and is used internally in "do-over" scenarios. Replay is also one reason determinism is so important to Temporal workflows. It is not about just being able to process new events but for queries and other capabilities, the entire workflow must be deterministic otherwise state would be inaccurate. An inaccurate state means we are stuck and cannot progress a workflow.

Below are the uses for replay:
- Testing with replayer to ensure workflow code is backward compatible
- Progressing workflow execution in event of unexpected failures
- Reset workflow execution
- Queries on completed workflows

## Replayer
The replayer is an SDK feature that uses replay for the purpose of testing. Performing replayer tests is the only way of ensuring code changes are backward-compatible with previous versions of a workflow and do not break determinism.
The Temporal SDK provides several replayer methods. In order to use the replayer, workflow history is required. The SDK attempts to make this easy and flexible providing various ways to get the workflow history.
![Replay](/assets/2023-02-02/replay.png)

What type of issues will replayer catch?
- Changing the Activity or Workflow Type
- Changing the order of Activities or Timers
- Adding new Activities, Timers, Signals, Queries in the past (before whatever is currently executing in a workflow)
- Potential non-determinism issues due to use of random numbers

### Output Workflow History
As mentioned there are various ways to get workflow history for replay. In this example we will save JSON output of workflow history to a file using tctl.

```bash
$ tctl wf show -w versioning-workflow --of history.json
```

### Create Replay Test
Depending on the replay SDK method chosen, arguments and inputs will of course change. In this case, since we have the workflow history as a JSON file, will use the ReplayWorkflowHistoryFromJSONFile method.

Below is code snippet showing how to run the replayer using workflow history from JSON file.
```go
func (s *replayTestSuite) TestReplayWorkflowHistoryFromFile() {
	replayer := worker.NewWorkflowReplayer()

	replayer.RegisterWorkflow(Workflow)

	err := replayer.ReplayWorkflowHistoryFromJSONFile(nil, "history.json")
	require.NoError(s.T(), err)
}
```

The full unit test can be found [here](https://github.com/ktenzer/temporal-demo-apps/blob/main/versioning/versioning_replay_test.go).

This example is a versioned Temporal workflow. As such, we are testing replay for each version to ensure, potential breaking changes are caught in our unit testing.

## Progressing Workflow Execution
As already mentioned, replay is an internal method used to resume making progress on a workflow execution. As an example, replay is triggered when a new worker takes over, in-progress work, from a worker that for whatever reason is no longer responsive. In such a scenario, we need to be careful about the changes made to running workflows, hence we have [workflow versioning strategies](https://keithtenzer.com/temporal/temporal_workflow_versioning/). We can always alter the future of a running workflow but changing the past will break determinism. Lets look at this in more detail with some concrete examples.

### Update Running Workflow in the Past
In this example we will execute some activities and then fire a timer to sleep a few minutes. We will then stop the worker and update the workflow code introducing a new activity before the timer, changing the past (from the running workflows point-of-view). Since we are changing or trying to re-write what already happened, a non-determinism error will be thrown.

Steps:
- Add a timer to workflow (recommend a few minutes)
- Run worker and start workflow
- Stop worker (ctrl-c)
- Add new activity before timer
- Run worker which will pickup the running workflow and trigger replay

#### Step 1: Add Timer
Ideally set the timer to a few minutes so you aren't waiting long.
```go
	err := workflow.ExecuteActivity(ctx, ActivityB).Get(ctx, &result)
	if err != nil {
		logger.Error("Activity failed.", "Error", err)
		return "", err
	}

	err = workflow.ExecuteActivity(ctx, ActivityC).Get(ctx, &result)
	if err != nil {
		logger.Error("Activity failed.", "Error", err)
		return "", err
	}

	// set timer for 5 minutes
	workflow.Sleep(ctx, time.Minute*5)
```

#### Step 2: Run Worker and Start Workflow
Start worker and run workflow with a starter or tctl.

#### Step 3: Stop Worker
Either CTRL-C or stop the worker process after the timer has fired.

#### Step 4: Add New Activity to Workflow (Past)
Add an additional activity before the timer in workflow.
```go
	err := workflow.ExecuteActivity(ctx, ActivityB).Get(ctx, &result)
	if err != nil {
		logger.Error("Activity failed.", "Error", err)
		return "", err
	}

	err = workflow.ExecuteActivity(ctx, ActivityC).Get(ctx, &result)
	if err != nil {
		logger.Error("Activity failed.", "Error", err)
		return "", err
	}

	// add new activity before timer
	err = workflow.ExecuteActivity(ctx, ActivityA).Get(ctx, &result)
	if err != nil {
		logger.Error("Activity failed.", "Error", err)
		return "", err
	}

	// set timer for 5 minutes
	workflow.Sleep(ctx, time.Minute*5)
```	

#### Step 5: Run Worker
The workflow is already running, the new worker will pickup the workflow and any currently or future executing activities.

#### Step 6: Observe Non-determinism Error
Since the newly added activity is in the past (from running workflow point-of-view), we are unable to progress the work because the workflow is now non-deterministic. The workflow will however remain running, we simply need to fix our code (remove the added activity), restart the worker and magically our workflow will complete.

![Non-Determinism](/assets/2023-02-02/non-determinism.png)

### Update Running Workflow in the Future
In this example we will execute some activities and then fire a timer to sleep a few minutes. We will then stop the worker and update the workflow code introducing a new activity after the timer. Since we are adding an activity after the timer we are not changing the past, only the future and as such, will not break determinism.

Steps:
- Add a timer to workflow (recommend a few minutes)
- Run worker and start workflow
- Stop worker (ctrl-c)
- Add new activity before timer
- Run worker which will pickup the running workflow and trigger replay

#### Step 1: Add Timer
Ideally set the timer to a few minutes so you aren't waiting long.
```go
	err := workflow.ExecuteActivity(ctx, ActivityB).Get(ctx, &result)
	if err != nil {
		logger.Error("Activity failed.", "Error", err)
		return "", err
	}

	err = workflow.ExecuteActivity(ctx, ActivityC).Get(ctx, &result)
	if err != nil {
		logger.Error("Activity failed.", "Error", err)
		return "", err
	}

	// set timer for 5 minutes
	workflow.Sleep(ctx, time.Minute*5)
```

#### Step 2: Run Worker and Start Workflow
Start worker and run workflow with a starter or tctl.

#### Step 3: Stop Worker
Either CTRL-C or stop the worker process after the timer has fired.

#### Step 4: Add New Activity to Workflow
Add an additional activity after the timer in workflow.
```go
	err := workflow.ExecuteActivity(ctx, ActivityB).Get(ctx, &result)
	if err != nil {
		logger.Error("Activity failed.", "Error", err)
		return "", err
	}

	err = workflow.ExecuteActivity(ctx, ActivityC).Get(ctx, &result)
	if err != nil {
		logger.Error("Activity failed.", "Error", err)
		return "", err
	}

	// set timer for 5 minutes
	workflow.Sleep(ctx, time.Minute*5)

	// add new activity after timer
	err = workflow.ExecuteActivity(ctx, ActivityA).Get(ctx, &result)
	if err != nil {
		logger.Error("Activity failed.", "Error", err)
		return "", err
	}
```	

#### Step 5: Run Worker
The workflow is already running, the new worker will pickup the workflow and any currently or future executing activities.

#### Step 6: Observe Successful Workflow Completion
Even though we added an activity to a running workflow, it was done in the future (from the workflow point-of-view), hence we did not break determinism and our workflow completed.

![Workflow Completed](/assets/2023-02-02/workflow_completed.png)

## Reset Workflow Execution
Replay is used when a user decides to reset a workflow execution from a certain point. All of the parameters and event history (workflow state) are preserved thanks to replay. 
A user can provide either either an reset_type (first task, last task, last continue-as-new) or event_id. This allows user to specify exactly from which point the workflow will be reset. Everything before that point is replayed and everything after executed.
Lets look at a concrete example. In our versioning workflow we will reset from when the timer started. First we will need to know the event_id for the timer.

![Timer Event Id](/assets/2023-02-02/event_id.png)

Next, using tctl we will reset the workflow from the event_id 18.
```bash
$ tctl workflow reset -w versioning-workflow --reason "testing" --event_id 18
```

The workflow will be reset causing a new workflow execution (runId). Looking at event history of new workflow execution, we can see the point where the reset was performed. Again everything before that point is replayed and everything after is executed.

![Workflow Reset](/assets/2023-02-02/reset.png)

## Queries on Completed Workflows
Temporal provides the ability to query workflows. Workflows can of course be running or completed. In the case of completed, replay is used to provide user-defined query results for a given workflow when the workflow is no longer in cache.
In our example, we have defined a query handler that provides a query_type called "state". Using tctl we can query our workflow using the query_type. 
```bash
$ tctl workflow query -w versioning-workflow --query_type "state"
```

## Summary
In this article we discussed Temporal replay, one of the resilience feature of Temporal workflows. We provided examples of some of the use cases for replay: testing for backward compatibility, progressing workflows, querying user-defined query types and resetting a workflow from a previous point-in-time. Replay is Temporal's time machine. It is also our safety mechanism to ensure like in "Back to the Future", we don't alter the past, breaking the space-time continuum and landing us in an altered, potentially broken future. 

(c) 2023 Keith Tenzer




