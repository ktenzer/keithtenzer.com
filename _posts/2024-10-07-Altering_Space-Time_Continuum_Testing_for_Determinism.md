--- 
layout: single
title:  "Altering Space-Time Continuum: Testing for Determinism in Temporal Workflows"
categories:
- Temporal
tags:
- Temporal
- Durable Execution
- Replay
- Determinism
- Workflow Orchestration
---

![Temporal](/assets/2022-08-15/logo-temporal-with-copy.svg)
## Overview
Even time traveling has its rules. To prevent one from creating an alternative future reality, Temporal requires Workflow determinism. In this article we will look at how to test Temporal workflows and ensure they are deterministic as part of CI/CD.

### What is Workflow Determinism?
A Temporal Workflow is Deterministic, if it does the same thing, producing the same outputs, given the same inputs.

Workflow Determinism is only checked when a Workflow is replayed. In Temporal, workflows are replayed when Workflow state must be reconstructed. Workflow state must be reconstructed when a Workflow is in the middle of execution and its worker no longer has that Workflow cached. A Workflow that is in the middle of execution, can be evicted from the Worker cache (due to memory), or can be re-scheduled to a new worker because for example, its original worker died, or is having issues.

### What Causes Non-Determinism?
The causes of Non-Deterministic errors are from doing something non-deterministic, such as UUID generation in Workflow code, or by not properly versioning Workflow changes. For example, changing order of Workflow Activity execution in Workflow from (Activity1, Activity2, Activity3) to (Activity3, Activity2, Activity1) and then restarting workers, while Workflow execution is still occurring. Restarting workers, forces currently executing workflows to replay and if Activity order (for activities that already occurred) changed, a Non-Deterministic error is thrown.

### Testing for Non-Determinism
Thankfully, the Temporal SDK provides mechanisms for performing Workflow replay. Replaying a Workflow takes a pre-recorded Temporal Workflow event history and replays it, against a given Workflow code version. Before rolling out new Workflow code changes, it is strongly recommended to perform a replay test.

There are two ways you can replay a Workflow:
- Using the SDK WorkflowReplayer from the testing facility
- Directly inside the Worker itself

Using the WorkflowReplayer, allows for specifying the Workflow classes that should be replayed vs using Replay from within the Worker, which will test only the classes that are registered, to that specific Worker.

The below sample in Java, shows how to download event history and perform a replay using WorkflowReplayer.
```java
public class WorkflowReplayer {
  private static final Logger log = LoggerFactory.getLogger(WorkflowReplayer.class);

  public void replayWorkflowHistory(String workflowId, WorkflowClient workflowClient) throws Exception {
    WorkflowExecutionHistory history = getWorkflowHistory(workflowId, workflowClient);
    log.info("Replaying workflow id " + workflowId);
    log.info(Environment.getServerInfo().toString());

    try {
      io.temporal.testing.WorkflowReplayer.replayWorkflowExecution(history, Hello.HelloWorkflowImpl.class);
    } catch (Exception e) {
      throw new RuntimeException("Failed to replay workflow " + workflowId, e);
    }
  }

  private WorkflowExecutionHistory getWorkflowHistory(String workflowId, WorkflowClient workflowClient) {
    return workflowClient.fetchHistory(workflowId);
  }

  public static void main(String[] args) throws Exception{
    WorkflowServiceStubs service = WorkflowServiceStubs.newServiceStubs(TemporalClient.getWorkflowServiceStubs().getOptions());

    WorkflowClientOptions.Builder builder = WorkflowClientOptions.newBuilder();
    WorkflowClientOptions clientOptions = builder.setNamespace(Environment.getNamespace()).build();
    WorkflowClient client = WorkflowClient.newInstance(service, clientOptions);


    WorkflowReplayer workflowReplayer = new WorkflowReplayer();
    String workflowId = Environment.getWorkflowId();

    try {
      workflowReplayer.replayWorkflowHistory(workflowId, client);
      log.info("Replay test successful");
      System.exit(0);
    } catch (Exception e) {
      throw new RuntimeException("Failed to replay workflowId " + workflowId + " " + e);
    } finally {
      System.exit(1);
    }
  }
}
```

The below sample in Java, shows how to download event history and perform a replay from within Worker.
```java
public class TemporalWorkerSelfReplay {

    @SuppressWarnings("CatchAndPrintStackTrace")
    public static void main(String[] args) throws Exception {
        final String TASK_QUEUE = Environment.getTaskqueue();

        String workflowId = Environment.getWorkflowId();
        WorkflowServiceStubs service = WorkflowServiceStubs.newServiceStubs(TemporalClient.getWorkflowServiceStubs().getOptions());
        WorkflowClientOptions.Builder builder = WorkflowClientOptions.newBuilder();
        WorkflowClientOptions clientOptions = builder.setNamespace(Environment.getNamespace()).build();
        WorkflowClient client = WorkflowClient.newInstance(service, clientOptions);

        WorkflowReplayer workflowReplayer = new WorkflowReplayer();

        WorkflowExecutionHistory history = workflowReplayer.getWorkflowHistory(workflowId, client);

        WorkerFactory factory = WorkerFactory.newInstance(TemporalClient.get());

        Worker worker = factory.newWorker(TASK_QUEUE);
        worker.registerWorkflowImplementationTypes(Hello.HelloWorkflowImpl.class);
        worker.registerActivitiesImplementations(new Hello.HelloActivitiesImpl());

        worker.replayWorkflowExecution(history);
        System.out.println("Workflow replay for workflowId: " + workflowId + " completed successfully");

        factory.start();
        System.out.println("Worker started for task queue: " + TASK_QUEUE);
    }
}
```

### Temporal Workflow Replay in CI/CD
Taking things a step further, instead of relying on every developer to test their Workflow code for Non-Determinism, wouldn't it be better to ensure this critical test happens as part of CI/CD?

#### Step 1: Run a Workflow
First, let's run a Temporal Workflow using a k8s job. The below sample, will launch a Worker and start a Temporal HelloWorkflow Workflow using [v1](https://github.com/temporal-sa/temporal-replayer/blob/v1/src/main/java/io/temporal/samples/replay/Hello.java) code.

  ```bash
  $ kubectl create -f yaml/job.yaml -n namespace
  ```

  The below sample, shows how to use k8s job to execute a Workflow.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  labels:
    app.kubernetes.io/build: "1"
    app.kubernetes.io/component: worker
    app.kubernetes.io/name: temporal-hello-starter
    app.kubernetes.io/version: v1.0
  name: temporal-hello-starter
spec:
  template:
    spec:
      containers:
      - env:
        - name: TEMPORAL_WORKFLOW_ID
          value: HelloWorkflow
        - name: TEMPORAL_ADDRESS
          value: helloworld.sdvdw.tmprl.cloud:7233
        - name: TEMPORAL_TASK_QUEUE
          value: hello
        - name: TEMPORAL_NAMESPACE
          value: helloworld.sdvdw
        - name: TEMPORAL_CERT_PATH
          value: /etc/certs/tls.crt
        - name: TEMPORAL_KEY_PATH
          value: /etc/certs/tls.key
        name: temporal-hello-starter
        image: ktenzer/temporal-hello-starter:v1.0
        imagePullPolicy: Always
        volumeMounts:
        - mountPath: /etc/certs
          name: certs
      volumes:
      - name: certs
        secret:
          defaultMode: 420
          secretName: temporal-tls
      restartPolicy: Never
  backoffLimit: 4
  ```

#### Step 2: Run Worker Deployment with Replay Test (v1)
Using the above WorkflowReplayer sample, a docker image is generated and the replay test is performed, prior to any deployment rollout. The replay test **succeeds** as the [v1 code path](https://github.com/temporal-sa/temporal-replayer/blob/v1/src/main/java/io/temporal/samples/replay/Hello.java#L113) is the same as the pre-recorded event history from step 1.

```bash
$ kubectl -f yaml/deployment-v1.yaml -n namespace
```

The below sample, shows how to use an init container within a deployment for executing replay test against v1 code. 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/build: "1"
    app.kubernetes.io/component: worker
    app.kubernetes.io/name: temporal-hello-worker
    app.kubernetes.io/version: v1.0
  name: temporal-hello-worker
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/component: worker
      app.kubernetes.io/name: temporal-hello-worker
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app.kubernetes.io/build: "1"
        app.kubernetes.io/component: worker
        app.kubernetes.io/name: temporal-hello-worker
        app.kubernetes.io/version: v1.0
    spec:
      initContainers:
      - env:
        - name: TEMPORAL_WORKFLOW_ID
          value: HelloWorkflow
        - name: TEMPORAL_ADDRESS
          value: helloworld.sdvdw.tmprl.cloud:7233
        - name: TEMPORAL_TASK_QUEUE
          value: hello
        - name: TEMPORAL_NAMESPACE
          value: helloworld.sdvdw
        - name: TEMPORAL_CERT_PATH
          value: /etc/certs/tls.crt
        - name: TEMPORAL_KEY_PATH
          value: /etc/certs/tls.key
        name: temporal-replayer
        image: ktenzer/temporal-replayer:v1.0
        imagePullPolicy: Always
        securityContext:
          allowPrivilegeEscalation: false
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /etc/certs
          name: certs
      containers:
      - env:
        - name: TEMPORAL_ADDRESS
          value: helloworld.sdvdw.tmprl.cloud:7233
        - name: TEMPORAL_TASK_QUEUE
          value: hello
        - name: TEMPORAL_NAMESPACE
          value: helloworld.sdvdw
        - name: TEMPORAL_CERT_PATH
          value: /etc/certs/tls.crt
        - name: TEMPORAL_KEY_PATH
          value: /etc/certs/tls.key
        image: ktenzer/temporal-hello-worker:v1.0
        imagePullPolicy: Always
        name: temporal-hello-worker
        imagePullPolicy: Always
        securityContext:
          allowPrivilegeEscalation: false
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /etc/certs
          name: certs
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - name: certs
        secret:
          defaultMode: 420
          secretName: temporal-tls
```

#### Step 3: Run Worker Deployment with Replay Test (v2)
Using the above WorkflowReplayer sample, a docker image is generated and the replay test is performed, prior to any deployment rollout. The replay test **fails** as the [v2 code path](https://github.com/temporal-sa/temporal-replayer/blob/v2/src/main/java/io/temporal/samples/replay/Hello.java#L113) is not the same as the pre-recorded event history from step 1.

```bash
$ kubectl create -f yaml/deployment-v2.yaml -n namespace
```

The below sample, shows how to use an init container within a deployment for executing replay test against v2 code.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/build: "1"
    app.kubernetes.io/component: worker
    app.kubernetes.io/name: temporal-hello-worker
    app.kubernetes.io/version: v2.0
  name: temporal-hello-worker
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/component: worker
      app.kubernetes.io/name: temporal-hello-worker
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app.kubernetes.io/build: "1"
        app.kubernetes.io/component: worker
        app.kubernetes.io/name: temporal-hello-worker
        app.kubernetes.io/version: v2.0
    spec:
      initContainers:
      - env:
        - name: TEMPORAL_WORKFLOW_ID
          value: HelloWorkflow
        - name: TEMPORAL_ADDRESS
          value: helloworld.sdvdw.tmprl.cloud:7233
        - name: TEMPORAL_TASK_QUEUE
          value: hello
        - name: TEMPORAL_NAMESPACE
          value: helloworld.sdvdw
        - name: TEMPORAL_CERT_PATH
          value: /etc/certs/tls.crt
        - name: TEMPORAL_KEY_PATH
          value: /etc/certs/tls.key
        name: temporal-replayer
        image: ktenzer/temporal-replayer:v2.0
        imagePullPolicy: Always
        securityContext:
          allowPrivilegeEscalation: false
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /etc/certs
          name: certs
      containers:
      - env:
        - name: TEMPORAL_ADDRESS
          value: helloworld.sdvdw.tmprl.cloud:7233
        - name: TEMPORAL_TASK_QUEUE
          value: hello
        - name: TEMPORAL_NAMESPACE
          value: helloworld.sdvdw
        - name: TEMPORAL_CERT_PATH
          value: /etc/certs/tls.crt
        - name: TEMPORAL_KEY_PATH
          value: /etc/certs/tls.key
        image: ktenzer/temporal-hello-worker:v2.0
        imagePullPolicy: Always
        name: temporal-hello-worker
        imagePullPolicy: Always
        securityContext:
          allowPrivilegeEscalation: false
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /etc/certs
          name: certs
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - name: certs
        secret:
          defaultMode: 420
          secretName: temporal-tls
```

For more details, refer to [Temporal Replayer](https://github.com/temporal-sa/temporal-replayer).

## Summary 
In this article we discussed the importance of Temporal Workflow Determinism. We looked at the causes for Non-determinism in Temporal workflows. Finally, leveraging the Java SDK, we showed how to write a replay test and even integrate it into CI/CD using a k8s deployment. 

(c) 2024 Keith Tenzer




