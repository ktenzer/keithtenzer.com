--- 
layout: single
title:  "Temporal Worker Versioning"
categories:
- Temporal
tags:
- Certificates
- mTLS
- Kubernetes
---

![Temporal](/assets/2022-08-15/logo-temporal-with-copy.svg)
## Overview
In this article we will cover the new feature, Worker Versioning. First, as of the writing of this article, Worker Versioning is in private beta and not yet GA. Worker Versioning is a new versioning strategy, allowing Task Queues and workers to be versioned using Build Ids. Worker Versioning should be used for short lived workflows and replace the older, Task Queue based versioning strategy. For long running workflows, Worker Versioning can certainly be used, however depending on requirements patch based or workflow name based versioning might still be a better fit.

## How does it work?
Worker Versioning matches a Task Queue version to a worker version. Versions are called Build IDs and can be grouped into Build Sets. Either the CLI or SDK can be used to create Build IDs and Build Sets. When a workflow is started, it is tagged with the default Build Id for the given Task Queue. In addition, workers must be assigned a Build ID and can only receive Workflow Tasks which match their Build ID tag.

![Worker Versioning](/assets/2023-09-06/version.png)

## Build Sets and Build Ids
A Build Set is one or more compatible Build Ids within a Task Queue. Each Build Set has a default Build Id. In addition the Task Queue itself has a default Build Set. Creating a new Build Id will, by default, result in that Build Id being the default for both the Build Set and Task Queue. It is possible to promote a Build Id to become the default, for example, if there is a regression and we want to go back to a previous Build Id.

## Managing Build Sets and Build Ids
Let's look at some examples, using the CLI for managing Build IDs. In the examples, we will create three Build IDs: 1.0, 1.1 and 2.0. The 1.0 and 1.1 Build IDs will be compatible, however 2.0 will not be compatible. In addition we will show how to manually promote a Build Id going from 2.0 back to 1.1.

#### Create first build-id and version
```
temporal task-queue update-build-ids add-new-default --task-queue default --build-id 1.0 --env prod
```

#### Create a second compatible build-id and version
```
$ temporal task-queue update-build-ids add-new-compatible --task-queue default --existing-compatible-build-id 1.0 --build-id 1.1 --env prod
```

#### Show Build IDs for a task-queue
```
$ temporal task-queue get-build-ids --task-queue default --env prod
  BuildIds   DefaultForSet  IsDefaultSet  
  [1.0 1.1]            1.1  true 

```

#### Show reachability for build-id 1.0
Build reachability shows if a Build Id will be used by any workflows. Here we can see the 1.0 build-id now has zero reachability.

```
$ temporal task-queue get-build-id-reachability --build-id 1.0 --env prod 
  BuildId  TaskQueue   Reachability   
      1.0  default    []
```

#### Show reachability for build-id 1.1
Looking at the 1.1 Build Id, we see that it will be applied to both new and existing (already running) workflows.

```
$ temporal task-queue get-build-id-reachability --build-id 1.1 --env prod 
  BuildId  TaskQueue  Reachability           
      1.1  default    [NewWorkflows                   
                      ExistingWorkflows]
```

#### Create a third non-compatible Build Id and version
```
$ temporal task-queue update-build-ids add-new-default --task-queue default --build-id 2.0 --env prod
```

#### Show Build IDs for a task-queue
The new 2.0 Build Id is automatically the Task Queue default and also the default for this new set.

```
$ temporal task-queue get-build-ids --task-queue default --env prod
  BuildIds   DefaultForSet  IsDefaultSet  
  [1.0 1.1]            1.1  false
  [2.0]                2.0  true  

```

#### Promote the 1.1 Build Id to default
Now let’s assume a regression with the 2.0 Build Id. We can promote the 1.1 Build Id to be the new default for the Task Queue.

```
$ temporal task-queue update-build-ids promote-set --task-queue default --build-id 1.1 --env prod
```

#### Show Build IDs for a task-queue
We can verify that the 1.1 version is in fact the default for the Task Queue.

```
$ temporal task-queue get-build-ids --task-queue default --env prod
  BuildIds   DefaultForSet  IsDefaultSet
  [2.0]                2.0  false      
  [1.0 1.1]            1.1  true
```



## Worker Configuration

Setting a Build Id for a Task Queue is only half of the equation. To use worker versioning, a Build Id should be assigned to the worker. This is done by configuring the worker options. Below is an example of setting the Build Id to 2.0 in Go. Under the resources section, there is a link to the official documentation which has documentation for every SDK.

```
...
    workerOptions := worker.Options{
        BuildID:                 "2.0",
        UseBuildIDForVersioning: true,
    }

    w := worker.New(c, os.Getenv("TEMPORAL_TASK_QUEUE"), workerOptions)
...  
```
## Limitations
The main limitations are 100 total Build IDs and 10 Build Sets on a single Task Queue. In short lived workflows, these limits should be of no concern, as versions will quickly be obsolete and no longer in use. However, long running workflows could hit some of these limits depending on requirements.


## Resources
[Official Worker Versioning Documentation for all SDKs](https://docs.temporal.io/workers#worker-versioning)

[Other Temporal Versioning Strategies](https://keithtenzer.com/temporal/temporal_workflow_versioning/)


## Summary
In this article we discussed the new worker versioning feature. We discussed not only how it works but also stepped through some examples of how to use the new worker versioning. This is a big step for Temporal as it greatly simplifies versioning, especially for short-lived workflows.

(c) 2023 Keith Tenzer




