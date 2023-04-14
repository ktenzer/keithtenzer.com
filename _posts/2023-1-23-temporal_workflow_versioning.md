--- 
layout: single
title:  "Altered Futures: Temporal Workflow Versioning"
categories:
- Temporal
tags:
- Temporal
- Workflows
- Versioning
---

![Temporal](/assets/2022-08-15/logo-temporal-with-copy.svg)
## Overview
In this article we will discuss Temporal workflow versioning. Since workflows can run a really long time or even indefinitely, versioning of workflows is not always a trivial task. After all, how do you alter a workflow while its still running?
In addition Temporal workflows are deterministic, changing of input parameters, order of activities or using random numbers, dates or even uuids in workflow code can break determinism, causing a non-deterministic error to be thrown.
Thankfully there are several approaches to workflow versioning depending on the objectives:
- Workflow code based versioning
- Taskqueue based versioning
- Workflow name based versioning

## Workflow Code Based Versioning
Workflow code based versioning involves branching your workflow code. It allows you to maintain versioning changes within the workflow code itself. The implementation differs slightly depending on which SDK is being used. Both Go and Java SDKs provide GetVersion APIs while other SDKs such as Typescript or Python provide a Patch. Here we will discuss using the GetVersion mechanism. Essentially your original workflow version is the default version. When the workflow is executed the version is recorded as a marker in the workflow event history. A search attribute, TemporalChangeVersion is upsertted so workflows with those changes are easy to search. You are able to increment the workflow version and use if-else block to implement what happens within a specific version of your workflow. This can sound very confusing so below we will walk-through the progression of releasing multiple workflow versions. 

### Default Version Example
In our original version we will simply execute an activity called ActivityA.
```go
err := workflow.ExecuteActivity(ctx, ActivityA).Get(ctx, &result)
if err != nil {
	logger.Error("Activity failed.", "Error", err)
	return "", err
}
```

### Version 1 Example
Instead of executing ActivityA we now want to execute ActivityB.
```go
v := workflow.GetVersion(ctx, "Version1", workflow.DefaultVersion, 1)
if v == workflow.DefaultVersion {
	err := workflow.ExecuteActivity(ctx, ActivityA).Get(ctx, &result)
	if err != nil {
		logger.Error("Activity failed.", "Error", err)
		return "", err
	}
} else {
	err := workflow.ExecuteActivity(ctx, ActivityB).Get(ctx, &result)
	if err != nil {
		logger.Error("Activity failed.", "Error", err)
		return "", err
	}
}
```

### Version 2 Example
Instead of executing Activity B we now want to execute ActivityB and ActivityC.
```go
v := workflow.GetVersion(ctx, "Version2", workflow.DefaultVersion, 2)
if v == workflow.DefaultVersion {
	err := workflow.ExecuteActivity(ctx, ActivityA).Get(ctx, &result)
	if err != nil {
		logger.Error("Activity failed.", "Error", err)
		return "", err
	}
} else if v == 1 {
	err := workflow.ExecuteActivity(ctx, ActivityB).Get(ctx, &result)
	if err != nil {
		logger.Error("Activity failed.", "Error", err)
		return "", err
	}
} else {
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
}
```

### Searching for workflow versions
A key to understanding when we can remove or consolidate our workflow version code is to find out if all running workflows for a given version are completed (no longer running). As mentioned above in Go the search attribute TemporalChangeVersion is automatically upsertted. It is planned to provide this in other languages but as of now you would need to manually upsert this attribute yourself but that is also simple enough.
Examining workflow history we can see the TemporalChangeVersion was upsertted.

![Upsert](/assets/2023-1-23/upsert.png)

Using the TemporalChangeVersion search attribute we can now look for workflows that match a specific version of our workflow code. 

![Search](/assets/2023-1-23/search.png)

Finally we can remove unneeded versions from our original code block, keeping our code clean.

## Task Queue Based Versioning
Task queue based versioning involves versioning the task queue names. Both the worker and any workflow starter need to pass a task queue name along with a workflow definition. The worker accomplishes this when instantiating a new worker object.

The name of the task queue is versioning. 
```go
w := worker.New(c, "versioning", worker.Options{})
```

In order to version using task queue you will want to create a new task queue for each version of the workflow.
```go
w := worker.New(c, "versioning-v1", worker.Options{})
```

You will also need to update any workflow starters which pass the task queue name through workflowOptions.
```go
workflowOptions := client.StartWorkflowOptions{
	ID:        "versioning-workflowId",
	TaskQueue: "versioning-v1",
}
```

In order to determine what versions of the workflow are running you can check the workflows workers. This will show the task queue (your version) and any workers that are polling against that task queue.

![Taskqueue](/assets/2023-1-23/taskqueue.png)

## Workflow Name Based Versioning
Workflow name based versioning is very similar to task queue based versioning. Both the worker and starter need to be updated whenever a new version is created as both load the workflow definition. Each time you create a new version you will simply create a new workflow definition file with a v1, v2, etc.

In the worker you would need to change the registration of the workflow and activities.
```go
w.RegisterWorkflow(versioning-v1.Workflow)
w.RegisterActivity(versioning-v1.ActivityA)
w.RegisterActivity(versioning-v1.ActivityB)
w.RegisterActivity(versioning-v1.ActivityC)
```	

In the starter you will need to update the ExecuteWorkflow call with the new workflow definition.
```go
we, err := c.ExecuteWorkflow(context.Background(), workflowOptions, versioning-v1.Workflow, "Temporal")
```

Of course just doing this alone makes it hard to determine what versions are running, in addition you might also consider reflecting the version in the workflow type or workflowId.

## Choosing a Versioning Strategy
Choosing the right versioning strategy depends greatly on your requirements and CI/CD processes. In general, if you don't want to micro-manage task queues/workers, change where starters are pointing and have the need to change future behavior of a running workflow without breaking compatibility of existing workflows, workflow code versioning will likely be the better approach. However, the other thing to keep in mind with workflow based versioning is it becomes quite unmanageable if you are to be maintaining many versions. You will definitely need to cleanup old versions that aren't in use to keep the code simple. Fortunately,  it is fairly simple, as demonstrated to query workflows and identify when a version is no longer needed.
Otherwise if you don't mind managing extra workers or task queues and don't need to change future behavior of running workflow, task queue or workflow name based versioning would be a simpler overall approach. 

Spencer Judge also gives some more details around advantages and disadvantages of the various [versioning strategies](https://community.temporal.io/t/workflow-versioning-strategies/6911).

## Summary
In this article we discussed the different strategies for versioning of Temporal workflows. We explored in detail workflow code, workflow name and task queue versioning. Finally, some general guidance in how to choose the appropriate strategy was provided.

(c) 2023 Keith Tenzer




