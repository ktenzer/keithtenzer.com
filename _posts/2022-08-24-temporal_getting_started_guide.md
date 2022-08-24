--- 
layout: single
title:  "Temporal Getting Started Guide"
categories:
- Temporal
tags:
- Temporal
- Opensource
- Workflows
- Activities
- Go
---

![Temporal](/assets/2022-08-15/logo-temporal-with-copy.svg)
## Overview
In this article we will walk through setup of a development environment for Temporal. There are of course, several ways you can run the Temporal server locally. You can use Docker compose, helm or an operator on minikube and temporalite. In addition you can also consume Temporal namespaces as-a-service via the Temporal cloud. I would recommend temporalite if you want to run the Temporal server disconnected on your laptop otherwise the Temporal cloud is definitely the best option. I will cover getting started with the Temporal cloud in a future post.

## Temporalite Install
Temporalite is a simple packaging of the Temporal server. It serializes data to local files, instead of a database for simplicity and less resource consumption. It is intended only for testing and not production use. In order to install it you need to have a Go environment to build the binary.

### Configure Go
#### Create directory structure for Go in your home directory.
<pre>$ mkdir -p ~/go/src/github.com</pre>

#### Install Go
<pre>$ sudo dnf install -y go</pre>

#### Update Profile with Go Environment
<pre>
$ vi ~/.bash_profile
export GOBIN=/home/username
export GOPATH=/home/username/src
export EDITOR=vim
</pre>

### Build Temporalite
#### Create directory for Temporal
<pre>$ mkdir ~/go/src/github.com/temporalio</pre>

#### Change directory to temporalio
<pre>$ cd temporalio</pre>

#### Clone temporalite GitHub repository
<pre>$ git clone https://github.com/temporalio/temporalite.git</pre>

#### Build temporalite binary
<pre>$ go install github.com/temporalio/temporalite/cmd/temporalite@latest</pre>

#### Copy temporalite binary to bin Directory
<pre>$ sudo cp /home/ktenzer/go/bin/temporalite /usr/local/bin</pre>

#### Create Directory for local Temporal Database
<pre>mkdir -p /home/ktenzer/.config/temporalite/db</pre>

#### Start temporalite
<pre>$ temporalite start --namespace default</pre>

## Build CLI
Now that we have the Temporal server running via temporalite we need to build the CLI.

#### Change directory to temporalio
<pre>$ cd ~/go/src/github.com/temporalio</pre>

#### Clone Temporal GitHub repository
<pre>$ git clone https://github.com/temporalio/temporal.git</pre>

#### Run tctl Makefile
<pre>$ make update-tctl</pre>

#### Copy tctl binary to bin directory
<pre>$ sudo cp ~/go/bin/tctl /usr/local/bin</pre>

## Start Temporal Workflow
In this case we will just run the helloworld sample from the many samples that Temporal provides.

#### Change directory to temporalio
<pre>$ cd ~/go/src/github.com/temporalio</pre>

#### Clone samples GitHub  repository
<pre>$ git clone https://github.com/temporalio/samples-go.git</pre>

#### Change directory to samples
<pre>$ cd samples-go</pre>

#### Start workflow
<pre>
$ go run helloworld/starter/main.go
2022/08/24 11:46:11 INFO  No logger configured for temporal client. Created default one.
2022/08/24 11:46:11 Started workflow WorkflowID hello_world_workflowID RunID b93ef614-e6df-4578-a31b-7ebd3f59df55
</pre>

Notice that the workflow starts but is waiting to execute. This is because there is no worker. The starter program will tell the Temporal server to execute a workflow however, since no worker is attached the Temporal server namespace, the workflow will be in running state until a worker becomes available.

The Temporal UI will also show that we are waiting for a worker.
![Temporal Workflow Started](/assets/2022-08-24/temporal_workflow_started.png)
  
## Start Temporal Worker
Now that we have our Temporal server and a workflow running all that is needed is a worker. We will start the corresponding helloworld worker.

<pre>$ go run helloworld/worker/main.go</pre>

The worker should immediately run and complete the workflow.
<pre>
2022/08/24 11:55:26 INFO  HelloWorld workflow completed. Namespace default TaskQueue hello-world WorkerID 7576@fedora@ WorkflowType Workflow WorkflowID hello_world_workflowID RunID b93ef614-e6df-4578-a31b-7ebd3f59df55 Attempt 1 result Hello Temporal!
</pre>

If you look at your starter it also has now returned and displays the workflow result returned from the workflow.
<pre>
2022/08/24 11:55:26 Workflow result: Hello Temporal!
</pre>

In the UI we can now also see the workflow has completed.
![Temporal Workflow Completed](/assets/2022-08-24/temporal_workflow_completed.png)

## View Workflow in CLI
Temporal provides the tctl CLI for interacting with the Temporal server.

#### List workflows
<pre>
$ tctl workflow list
  WORKFLOW TYPE |      WORKFLOW ID       |                RUN ID                | TASK QUEUE  | START TIME | EXECUTION TIME | END TIME  
  Workflow      | hello_world_workflowID | b93ef614-e6df-4578-a31b-7ebd3f59df55 | hello-world | 18:46:11   | 18:46:11       | 18:55:26  
</pre>

#### Show workflow details
<pre>
$ tctl workflow show --workflow_id hello_world_workflowID --run_id b93ef614-e6df-4578-a31b-7ebd3f59df55
   1  WorkflowExecutionStarted    {WorkflowType:{Name:Workflow},                                 
                                  ParentInitiatedEventId:0, TaskQueue:{Name:hello-world,         
                                  Kind:Normal}, Input:["Temporal"],                              
                                  WorkflowExecutionTimeout:0s, WorkflowRunTimeout:0s,            
                                  WorkflowTaskTimeout:10s, Initiator:Unspecified,                
                                  OriginalExecutionRunId:b93ef614-e6df-4578-a31b-7ebd3f59df55,   
                                  Identity:5846@fedora@,                                         
                                  FirstExecutionRunId:b93ef614-e6df-4578-a31b-7ebd3f59df55,      
                                  Attempt:1, FirstWorkflowTaskBackoff:0s,                        
                                  ParentInitiatedEventVersion:0}                                 
   2  WorkflowTaskScheduled       {TaskQueue:{Name:hello-world,                                  
                                  Kind:Normal},                                                  
                                  StartToCloseTimeout:10s,                                       
                                  Attempt:1}                                                     
   3  WorkflowTaskStarted         {ScheduledEventId:2, Identity:7576@fedora@,                    
                                  RequestId:3c597463-65e4-4d61-b014-484634f32c37,                
                                  SuggestContinueAsNew:false, HistorySizeBytes:0}                
   4  WorkflowTaskCompleted       {ScheduledEventId:2, StartedEventId:3,                         
                                  Identity:7576@fedora@,                                         
                                  BinaryChecksum:0990e4a32efe7cd9c6c127b9cb51ecfc}               
   5  ActivityTaskScheduled       {ActivityId:5,                                                 
                                  ActivityType:{Name:Activity},                                  
                                  TaskQueue:{Name:hello-world,                                   
                                  Kind:Normal},                                                  
                                  Input:["Temporal"],                                            
                                  ScheduleToCloseTimeout:0s,                                     
                                  ScheduleToStartTimeout:0s,                                     
                                  StartToCloseTimeout:10s,                                       
                                  HeartbeatTimeout:0s,                                           
                                  WorkflowTaskCompletedEventId:4,                                
                                  RetryPolicy:{InitialInterval:1s,                               
                                  BackoffCoefficient:2,                                          
                                  MaximumInterval:1m40s,                                         
                                  MaximumAttempts:0,                                             
                                  NonRetryableErrorTypes:[]}}                                    
   6  ActivityTaskStarted         {ScheduledEventId:5, Identity:7576@fedora@,                    
                                  RequestId:9a1da09d-2b00-4c46-a945-9e24adc10c04,                
                                  Attempt:1}                                                     
   7  ActivityTaskCompleted       {Result:["Hello                                                
                                  Temporal!"],                                                   
                                  ScheduledEventId:5,                                            
                                  StartedEventId:6,                                              
                                  Identity:7576@fedora@}                                         
   8  WorkflowTaskScheduled       {TaskQueue:{Name:fedora:e78b4021-eed7-4cb9-880e-9a7354471838,  
                                  Kind:Sticky}, StartToCloseTimeout:10s, Attempt:1}              
   9  WorkflowTaskStarted         {ScheduledEventId:8, Identity:7576@fedora@,                    
                                  RequestId:8cc430d3-3d73-4261-961e-7dacd1b13005,                
                                  SuggestContinueAsNew:false, HistorySizeBytes:0}                
  10  WorkflowTaskCompleted       {ScheduledEventId:8, StartedEventId:9,                         
                                  Identity:7576@fedora@,                                         
                                  BinaryChecksum:0990e4a32efe7cd9c6c127b9cb51ecfc}               
  11  WorkflowExecutionCompleted  {Result:["Hello                                                
                                  Temporal!"],                                                   
                                  WorkflowTaskCompletedEventId:10} 
</pre>

## Summary
In this article we discussed several options for running the Temporal server, including using the Temporal cloud. We walked through the steps; deploying a quick local development environment using temporalite. Finally we showed how to execute and get workflow details using both the Temporal CLI and UI. Best of luck in your journey to more reliable applications with Temporal! If you have any questions reach out on [Temporal community slack](https://temporal.io/slack).

(c) 2022 Keith Tenzer




