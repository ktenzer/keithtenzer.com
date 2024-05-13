--- 
layout: single
title:  "Temporal Batch Operations Primer"
categories:
- Temporal
tags:
- Workflow
- Batch Operations
- Batch Signal
- Batch Reset
- Batch Reset
- Temporal Cloud
- Automation
---

![Temporal](/assets/2022-08-15/logo-temporal-with-copy.svg)
## Overview
Temporal Cloud can run billions of workflows. As such being able to perform batch operations for of workflows becomes critical, especially when issues arise. Temporal provides three types of batch operations: terminate, reset and signal. In addition Temporal provides the ability to list and terminate batch jobs.

## General Guidelines
Batch operations use a visibility query to determine the scope (workflows), against which the operation shall be performed. If using customer search attributes in visibility query, it is important to understand that as they are eventually consistent, some workflows started just before batch operation may not be included as the custom search attribute was not yet propagated. In such cases it may require running a second batch operation or not using custom search attributes if it is a concern.

Batch operations have a default setting of 50 operations a second. This number must be increased at Temporal Cloud namespace and SDK level.
```
{
  "worker.batcherRPS": 200
}
```

```
setMaxOperationsPerSecond(200)
```

By default batch operations are limited to 1 running per namespace. This can be increased but must be done at Temporal Cloud level.
```
{
  "frontend.MaxConcurrentBatchOperationPerNamespace": 3,
}
```

## Terminate Batch Operation
The batch terminate operation allows for terminating multiple workflows. A batch terminate operation is typically used when many workflows need to be terminated at once. For example, due to a downstream failure or even a runaway process, creating workflows inadvertently. We should limit the visibility query scope to be only running workflows. Otherwise we will needlessly send terminations to already completed workflows.

The example below shows how to use the batch terminate operation in the Java SDK.

```java
private ResponseEntity runBatchTerminate() {
    String jobId = UUID.randomUUID().toString();
    client.getWorkflowServiceStubs()
            .blockingStub()
            .startBatchOperation(
                    StartBatchOperationRequest.newBuilder()
                            .setNamespace(client.getOptions().getNamespace())
                            .setJobId(jobId)
                            .setVisibilityQuery("ExecutionStatus='Running' AND WorkflowType='SampleWorkflow'")
                            .setReason("Terminating all running executions")
                            .setTerminationOperation(BatchOperationTermination.newBuilder()
                                    .setIdentity(client.getOptions().getIdentity())
                                    .build())
                            .build());

    return new ResponseEntity<>(new SampleResult("Started batch operation to terminate all running executions: " + jobId), HttpStatus.OK);
} 
```

## Signal Batch Operation
The signal batch operation allows you to signal multiple workflows. A Batch signal operation is typically used when many workflows should be signaled in order to block/unblock (pause/resume) workflows. 
As signals can only be sent to running workflows, it is important we limit visibility query scope to be only running workflows. Otherwise we will needlessly send signals to already completed workflows.

The example below shows how to use the batch signal operation in the Java SDK.
```java
private ResponseEntity runBatchSignal() {
    String jobId = UUID.randomUUID().toString();
    client.getWorkflowServiceStubs()
            .blockingStub()
            .startBatchOperation(
                    StartBatchOperationRequest.newBuilder()
                            .setNamespace(client.getOptions().getNamespace())
                            .setJobId(jobId)
                            .setVisibilityQuery("ExecutionStatus='Running' AND WorkflowType='SampleWorkflow'")
                            .setReason("Retrying all paused executions")
                            .setSignalOperation(BatchOperationSignal.newBuilder()
                                    .setSignal("resume")
                                    .setIdentity(client.getOptions().getIdentity())
                                    .build())
                            .build());

    return new ResponseEntity<>(new SampleResult("Started batch operation to retry all paused executions: " + jobId), HttpStatus.OK);
}
```

## Reset Batch Operation
The reset batch operation allows you to reset multiple workflows. In the event that many workflows are failed or stuck, either intentionally or due to your business logic; you can easily reset them either to the beginning, the last workflow task (where it left off) or even from a specific workflow task. While you can reset any workflow, we should still consider targeting the visibility scope to a specific workflow type and execution status.

The below Java example will perform a batch reset, resetting all failed workflows of the workflow type SampleWorkflow to the last workflow task.
```java
private ResponseEntity runResetTerminate(String visibilityQuery, String signalName) {
    String jobId = UUID.randomUUID().toString();
    client.getWorkflowServiceStubs()
            .blockingStub()
            .startBatchOperation(
                    StartBatchOperationRequest.newBuilder()
                            .setNamespace(client.getOptions().getNamespace())
                            .setJobId(jobId)
                            .setVisibilityQuery("ExecutionStatus='Failed' AND WorkflowType='SampleWorkflow'")
                            .setReason("Retrying all paused executions")
                            .setResetOperation(BatchOperationReset.newBuilder()
                                    .setIdentity(client.getOptions().getIdentity())
                                    .setResetType(ResetType.RESET_TYPE_LAST_WORKFLOW_TASK)
                                    .build())
                            .build());

    return new ResponseEntity<>(new SampleResult("Started batch operation to reset all failed executions: " + jobId), HttpStatus.OK);
}
```

## List Batch Operation
The list batch operations API show the status of any running batch operations. Below the Temporal Cloud UI is shown to view a batch operation. Notice the operation id, type and results.

![Batch List](/assets/2024-05-10/batch_list_example.png)

The example below shows how to list batch operations using the Java SDK.

```java
private List<BatchOperationInfo> getBatchInfo(WorkflowClient client, ByteString nextPageToken, List<BatchOperationInfo> info) {
    if (info == null) {
        info = new ArrayList<>();
    }
    ListBatchOperationsResponse res = client.getWorkflowServiceStubs().blockingStub().listBatchOperations(ListBatchOperationsRequest.newBuilder()
            .setNamespace(client.getOptions().getNamespace())
            .build());
    info.addAll(res.getOperationInfoList());
    if (res.getNextPageToken() != null && res.getNextPageToken().size() > 0) {
        return getBatchInfo(client, res.getNextPageToken(), info);
    } else {
        return info;
    }
}
```

## Terminate Batch Operation
The terminate batch operation API provides ability to cancel or terminate a running batch operation. For example, there could be issue with visibility query where the wrong workflows are being targeted.

The example below shows how to terminate a batch job using the Java SDK.

```java
try {
    client.getWorkflowServiceStubs().blockingStub().stopBatchOperation(
            StopBatchOperationRequest.newBuilder()
                    .setNamespace(client.getOptions().getNamespace())
                    .setReason("stopping batch")
                    .setIdentity(client.getOptions().getIdentity())
                    .setJobId(in.getJobId())
                    .build()
    );
} catch (Exception e) {
    e.printStackTrace();
}
```

## Summary
In this article we discussed the different types of batch operations Temporal Cloud offers. Batch operations are very powerful and let you take action on many, many workflows at the same time while also providing an important throttling mechanism. They are powered by the visibility API, enabling a query language to narrow down exactly what workflows should be targeted. Examples of how to run batch terminate, signal, and reset were provided in Java. Finally, it was also demonstrated how batch operations could be managed and even themselves, terminated.

(c) 2024 Keith Tenzer




