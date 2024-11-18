--- 
layout: single
title:  "Temporal Fundamentals Part V: Workflow Patterns"
categories:
- Temporal
tags:
- Workflow Patterns
- Design Patterns
- Temporal
- Workflow
- Patterns
---

![Temporal](/assets/2022-08-15/logo-temporal-with-copy.svg)
## Overview
This is a six part series focused on Temporal fundamentals. It represents, in my words, what I have learned along the way and what I would've like to know on day one.

- [Temporal Fundamentals Part I: Basics](https://keithtenzer.com/temporal/Temporal_Fundamentals_Basics)
- [Temporal Fundamentals Part II: Concepts](https://keithtenzer.com/temporal/Temporal_Fundamentals_Concepts/)
- [Temporal Fundamentals Part III: Timeouts](https://keithtenzer.com/temporal/Temporal_Fundamentals_Timeouts/)
- [Temporal Fundamentals Part IV: Workflows](https://keithtenzer.com/temporal/Temporal_Fundamentals_Workflows/)
- [Temporal Fundamentals Part V: Workflow Patterns](https://keithtenzer.com/temporal/Temporal_Fundamentals_Workflow_Patterns/)
- Temporal Fundamentals Part VI: Workers

This article will explore some of the more common Temporal design patterns. There are many reusable solutions within Temporal Workflows that will help speedup development, ensure problems are avoided, improve code readability and get the most out of leveraging Temporal.

## Workflow Patterns
Below is a list of patterns covered:
- [Workflow and Activity Compensation](/temporal/Temporal_Fundamentals_Workflow_Patterns/#workflow-and-activity-composition)
- [SAGA](/temporal/Temporal_Fundamentals_Workflow_Patterns/#saga)
- [Polling](/temporal/Temporal_Fundamentals_Workflow_Patterns/#polling)
- [Batch Processing](/temporal/Temporal_Fundamentals_Workflow_Patterns/#batch-processing)
- [Actor Workflows](/temporal/Temporal_Fundamentals_Workflow_Patterns/#actor-workflows)
- [Request and Response](/temporal/Temporal_Fundamentals_Workflow_Patterns/#request-and-response)
- [Retry and Error Handling](/temporal/Temporal_Fundamentals_Workflow_Patterns/#retry-and-error-handling)
- [Activity Timeouts and Heartbeats](/temporal/Temporal_Fundamentals_Workflow_Patterns/#activity-timeouts-and-heartbeats)
- [Versioning](/temporal/Temporal_Fundamentals_Workflow_Patterns/#versioning)
- [Cancellation and Compensation](/temporal/Temporal_Fundamentals_Workflow_Patterns/#cancellation-and-compensation)

As we look at specific patterns, we will show both Workflow and Activity code. 

## Workflow and Activity Composition
Temporal enables breaking business logic down, into smaller reusable pieces. In Temporal, Activities are components that perform concrete tasks. More specifically, anything that can fail, needs to be inside Activity. Workflows orchestrate Activities, managing their sequence and directing the overall business process to completion.

The below code shows relationship between Workflow and Activity.

### Activity
```java
@ActivityInterface
public interface GreetingActivities {
    @ActivityMethod
    String composeGreeting(String greeting, String name);
}

public static class GreetingActivitiesImpl implements GreetingActivities {
    @Override
    public String composeGreeting(String greeting, String name) {
        return greeting + " " + name + "!";
    }
}
```

### Workflow
```java
@WorkflowInterface
public interface GreetingWorkflow {
    @WorkflowMethod
    String getGreeting(String name);
}

public class GreetingWorkflowImpl implements GreetingWorkflow {
    private final GreetingActivities activities = Workflow.newActivityStub(GreetingActivities.class);

    @Override
    public String getGreeting(String name) {
        // Call an activity within the workflow
        return activities.composeGreeting("Hello", name);
    }
}
```

## SAGA
Distributed SAGAs in Temporal are trivial, and amount to a simple try/catch with compensation. Since Temporal can recover, and resume execution from any failure, we only need to concern ourselves with business logic. This is a big reason why, so many organizations rely on Temporal for mission critical business processes, such as bookings, orders, payments, loans, subscriptions and more. 

The below code [sample](https://github.com/temporalio/samples-java/tree/main/core/src/main/java/io/temporal/samples/bookingsaga), shows how to create a SAGA, and run compensation using a multi-step booking as example.

```java
public void bookVacation(BookingInfo info) {
   Saga saga = new Saga(new Saga.Options.Builder().build());
   try {
       saga.addCompensation(activities::cancelHotel, info.getClientId());
       activities.bookHotel(info);

       saga.addCompensation(activities::cancelFlight, info.getClientId());
       activities.bookFlight(info);

       saga.addCompensation(activities::cancelExcursion, 
                            info.getClientId());
       activities.bookExcursion(info);
   } catch (TemporalFailure e) {
       saga.compensate();
       throw e;
   }
}
```

## Polling
It is often the case, that Workflows may need to wait for external processing. There different types of polling: frequent, infrequent or even periodic. Each have different implementations.

### Frequent
Frequent polling should be done inside the Activity, it is very important to Heartbeat as this will be a long running Activity.

The below code [sample](https://github.com/temporalio/samples-java/tree/main/core/src/main/java/io/temporal/samples/polling/frequent), shows how to implement polling inside an Activity.

#### Activity
```java
@ActivityInterface
public interface PollingActivities {
  String doPoll();
}

public class FrequentPollingActivityImpl implements PollingActivities {
  private final TestService service;
  private static final int POLL_DURATION_SECONDS = 1;

  public FrequentPollingActivityImpl(TestService service) {
    this.service = service;
  }

  @Override
  public String doPoll() {
    ActivityExecutionContext context = Activity.getExecutionContext();

    while (true) {
      try {
        return service.getServiceResult();
      } catch (TestService.TestServiceException e) {
        // service "down" we can log
      }

      // heart beat and sleep for the poll duration
      try {
        context.heartbeat(null);
      } catch (ActivityCompletionException e) {
        // activity was either cancelled or workflow was completed or worker shut down
        throw e;
      }
      sleep(POLL_DURATION_SECONDS);
    }
  }

  private void sleep(int seconds) {
    try {
      Thread.sleep(TimeUnit.SECONDS.toMillis(seconds));
    } catch (InterruptedException e) {
      Thread.currentThread().interrupt();
      throw new RuntimeException(e);
    }
  }
}
```

### Infrequent
For infrequent polling, we can simply use Activity retries and the configurable backoff coefficient, increasing time between retries, configured at the Workflow level.

The below code [sample](https://github.com/temporalio/samples-java/tree/main/core/src/main/java/io/temporal/samples/polling/infrequent), shows how to configure polling using Activity retry options.

#### Workflow
```java
@WorkflowInterface
public interface PollingWorkflow {
  @WorkflowMethod
  String exec();
}

public class InfrequentPollingWorkflowImpl implements PollingWorkflow {
  @Override
  public String exec() {
    ActivityOptions options =
        ActivityOptions.newBuilder()
            // Set activity StartToClose timeout (single activity exec), does not include retries
            .setStartToCloseTimeout(Duration.ofSeconds(2))
            .setRetryOptions(
                RetryOptions.newBuilder()
                    .setBackoffCoefficient(1)
                    .setInitialInterval(Duration.ofSeconds(60))
                    .build())
            .build();
    // create our activities stub and start activity execution
    PollingActivities activities = Workflow.newActivityStub(PollingActivities.class, options);
    return activities.doPoll();
  }
}
```

## Batch Processing
In Temporal, batch processing is a pattern for executing background operations over a large set of Workflows, all at once. 

Typical use cases are:
- Bulk Signals/Cancellations/Terminations of Workflows
- Data Analysis or transforming raw data into something useful for modelling 
- Data Processing for reporting, warehousing, payroll processing and more

Batch processing usually involves grouping data together, or a sliding window. For a large dataset, we would recommend fan-out, using a Child Workflow for every group, and within each Workflow, every item would be processed by an activity. If items can share same retry policy, compensation logic, failure handling, it might make sense to group multiple items into same Activity.

Batch processing can be implemented as a Heartbeating Activity, Iterator or or even Sliding Window.

The below code [sample](https://github.com/temporalio/samples-java/tree/main/core/src/main/java/io/temporal/samples/batch/iterator), shows a iterator batch use case, where batching is done using Child Workflows.

### Activity
```java
@ActivityInterface
public interface BatchActivities {
    @ActivityMethod
    void processData(String data);
}

public static class BatchActivitiesImpl implements BatchActivities {
    @Override
    public void processData(String data) {
        System.out.println("Processing data: " + data);
        // Add business logic here
    }
}
```

### Workflow
```java
@WorkflowInterface
public interface IteratorBatchWorkflow {
  @WorkflowMethod
  int processBatch(int pageSize, int offset);
}

public final class IteratorBatchWorkflowImpl implements IteratorBatchWorkflow {

  private final RecordLoader recordLoader =
      Workflow.newActivityStub(
          RecordLoader.class,
          ActivityOptions.newBuilder().setStartToCloseTimeout(Duration.ofSeconds(5)).build());

  /** Stub used to continue-as-new. */
  private final IteratorBatchWorkflow nextRun =
      Workflow.newContinueAsNewStub(IteratorBatchWorkflow.class);

  @Override
  public int processBatch(int pageSize, int offset) {
    // Loads a page of records
    List<SingleRecord> records = recordLoader.getRecords(pageSize, offset);
    // Starts a child per record asynchrnously.
    List<Promise<Void>> results = new ArrayList<>(records.size());
    for (SingleRecord record : records) {
      // Uses human friendly child id.
      String childId = Workflow.getInfo().getWorkflowId() + "/" + record.getId();
      RecordProcessorWorkflow processor =
          Workflow.newChildWorkflowStub(
              RecordProcessorWorkflow.class,
              ChildWorkflowOptions.newBuilder().setWorkflowId(childId).build());
      Promise<Void> result = Async.procedure(processor::processRecord, record);
      results.add(result);
    }
    // Waits for all children to complete.
    Promise.allOf(results).get();

    // No more records in the dataset. Completes the workflow.
    if (records.isEmpty()) {
      return offset;
    }

    // Continues as new with the increased offset.
    return nextRun.processBatch(pageSize, offset + records.size());
  }
}
```

## Actor Workflows
In Temporal there is no constraint of time and as such, Workflows can live forever. Workflows can even encapsulate an entities behavior. Actor Workflows can send/receive messages, maintain state or even create new Actors.

Typical use cases are:
- Subscriptions
- Users
- Orders
- Shipments
- And much more...

The below code [sample](https://github.com/temporalio/subscription-workflow-project-template-java/tree/main), shows a example of leveraging the Actor pattern to build a subscription Workflow.

### Workflow
```java

@WorkflowInterface
public interface SubscriptionWorkflow {
  @WorkflowMethod
  void startSubscription(Customer customer);

  @SignalMethod
  void cancelSubscription();

  @SignalMethod
  void updateBillingPeriodChargeAmount(int billingPeriodChargeAmount);

  @QueryMethod
  String queryCustomerId();

  @QueryMethod
  int queryBillingPeriodNumber();

  @QueryMethod
  int queryBillingPeriodChargeAmount();
}

public class SubscriptionWorkflowImpl implements SubscriptionWorkflow {

  private int billingPeriodNum;
  private boolean subscriptionCancelled;
  private Customer customer;

  // Define our Activity options: setStartToCloseTimeout: maximum Activity Execution time after it was sent to a Worker
   
  private final ActivityOptions activityOptions =
      ActivityOptions.newBuilder().setStartToCloseTimeout(Duration.ofSeconds(5)).build();

  // Define subscription Activities stub
  private final SubscriptionActivities activities =
      Workflow.newActivityStub(SubscriptionActivities.class, activityOptions);

  @Override
  public void startSubscription(Customer customer) {
    // Set the Workflow customer
    this.customer = customer;

    // Send welcome email to customer
    activities.sendWelcomeEmail(customer);

    // Start the free trial period. User can still cancel subscription during this time
    Workflow.await(customer.getSubscription().getTrialPeriod(), () -> subscriptionCancelled);

    // If customer cancelled their subscription during trial period, send notification email
    if (subscriptionCancelled) {
      activities.sendCancellationEmailDuringTrialPeriod(customer);
      // We have completed subscription for this customer.
      // Finishing Workflow Execution
      return;
    }

    // Trial period is over, start billing until
    // we reach the max billing periods for the subscription
    // or sub has been cancelled
    while (billingPeriodNum < customer.getSubscription().getMaxBillingPeriods()) {

      // Charge customer for the billing period
      activities.chargeCustomerForBillingPeriod(customer, billingPeriodNum);

      // Wait 1 billing period to charge customer or if they cancel subscription
      // whichever comes first
      Workflow.await(customer.getSubscription().getBillingPeriod(), () -> subscriptionCancelled);

      // If customer cancelled their subscription send notification email
      if (subscriptionCancelled) {
        activities.sendCancellationEmailDuringActiveSubscription(customer);

        // We have completed subscription for this customer.
        // Finishing Workflow Execution
        break;
      }

      billingPeriodNum++;
    }

    // if we get here the subscription period is over
    // notify the customer to buy a new subscription
    if (!subscriptionCancelled) {
      activities.sendSubscriptionOverEmail(customer);
    }
  }

  @Override
  public void cancelSubscription() {
    subscriptionCancelled = true;
  }

  @Override
  public void updateBillingPeriodChargeAmount(int billingPeriodChargeAmount) {
    customer.getSubscription().setBillingPeriodCharge(billingPeriodChargeAmount);
  }

  @Override
  public String queryCustomerId() {
    return customer.getId();
  }

  @Override
  public int queryBillingPeriodNumber() {
    return billingPeriodNum;
  }

  @Override
  public int queryBillingPeriodChargeAmount() {
    return customer.getSubscription().getBillingPeriodCharge();
  }
}
```

### Client
```java
public class SubscriptionWorkflowStarter {

  // Task Queue name
  public static final String TASK_QUEUE = "SubscriptionsTaskQueue";
  // Base Id for all subscription Workflow Ids
  public static final String WORKFLOW_ID_BASE = "SubscriptionsWorkflow";

  // Define our Subscription, Let's say we have a trial period of 10 seconds and a billing period of 10 seconds.In real life this would be much longer. We also set the max billing periods to 24, and the billing cycle charge to 120

  public static Subscription subscription =
      new Subscription(Duration.ofSeconds(10), Duration.ofSeconds(10), 24, 120);

  public static void main(String[] args) {
    WorkflowServiceStubs service = WorkflowServiceStubs.newLocalServiceStubs();
    WorkflowClient client = WorkflowClient.newInstance(service);
    WorkerFactory factory = WorkerFactory.newInstance(client);
    Worker worker = factory.newWorker(TASK_QUEUE);
 
    worker.registerWorkflowImplementationTypes(SubscriptionWorkflowImpl.class);
    worker.registerActivitiesImplementations(new SubscriptionActivitiesImpl());

    // Start all the Workers registered for a specific Task Queue.
    factory.start();

    // List of our example customers
    List<Customer> customers = new ArrayList<>();

    // Create example customers
    for (int i = 0; i < 5; i++) {
      Customer customer =
          new Customer("First Name" + i, "Last Name" + i, "Id-" + i, "Email" + i, subscription);
      customers.add(customer);
    }

    // Create and start a new subscription Workflow for each of the example customers

    customers.forEach(
        customer -> {
          SubscriptionWorkflow workflow =
              client.newWorkflowStub(
                  SubscriptionWorkflow.class,
                  WorkflowOptions.newBuilder()
                      .setTaskQueue(TASK_QUEUE)
                      .setWorkflowId(WORKFLOW_ID_BASE + customer.getId())
                      .setWorkflowRunTimeout(Duration.ofMinutes(5))
                      .build());

          // Start Workflow Execution (async)
          WorkflowClient.start(workflow::startSubscription, customer);
        });
  }
}
```

## Request and Response
A common pattern in software development is request/response. Temporal provides the Workflow Update primitive, which can not only return data back to the caller but also handle validation. The primitive also supports UpdateWithStart so that a Workflow can be started, do some validation or processing, return back to caller and then continue with anything else asynchronously.

The below code [sample](https://github.com/temporalio/samples-java/blob/main/core/src/main/java/io/temporal/samples/hello/HelloUpdate.java), uses the Temporal Update primitive to send greetings. If the Update passes validation, the size of the message and queue are returned to the caller, otherwise an error is returned.

### Workflow
```java
  @WorkflowInterface
  public interface GreetingWorkflow {
    @WorkflowMethod
    List<String> getGreetings();

    @UpdateMethod
    int addGreeting(String name);

    @UpdateValidatorMethod(updateName = "addGreeting")
    void addGreetingValidator(String name);

    @SignalMethod
    void exit();
  }

  public static class GreetingWorkflowImpl implements GreetingWorkflow {

    // messageQueue holds up to 10 messages (received from updates)
    private final List<String> messageQueue = new ArrayList<>(10);
    private final List<String> receivedMessages = new ArrayList<>(10);
    private boolean exit = false;

    private final HelloActivity.GreetingActivities activities =
        Workflow.newActivityStub(
            HelloActivity.GreetingActivities.class,
            ActivityOptions.newBuilder().setStartToCloseTimeout(Duration.ofSeconds(2)).build());

    @Override
    public List<String> getGreetings() {

      while (true) {
        // Block current thread until the unblocking condition is evaluated to true
        Workflow.await(() -> !messageQueue.isEmpty() || exit);
        if (messageQueue.isEmpty() && exit) {
          return receivedMessages;
        }
        String message = messageQueue.remove(0);
        receivedMessages.add(message);
      }
    }

    @Override
    public int addGreeting(String name) {
      if (name.isEmpty()) {
        throw ApplicationFailure.newFailure("Cannot greet someone with an empty name", "Failure");
      }
      // Updates can mutate workflow state like variables or call activities
      messageQueue.add(activities.composeGreeting("Hello", name));
      // Updates can return data back to the client
      return receivedMessages.size() + messageQueue.size();
    }

    @Override
    public void addGreetingValidator(String name) {
      if (receivedMessages.size() >= 10) {
        throw new IllegalStateException("Only 10 greetings may be added");
      }
    }

    @Override
    public void exit() {
      exit = true;
    }
  }
```

## Retry and Error Handling
In Temporal, the default behavior is to progress a Workflow to completion. However, for certain errors, we may want to either change the behavior or even fail the Workflow. A retry policy is applied to Activities and by default it is unlimited. If we limit retries, and those retries are exhausted, the Temporal Workflow will fail. Similarly, depending on error condition, we may want to fail a Workflow explicitly. We can catch and re-throw errors, or even specify certain errors a Non-Retryable in our retry policy.  

The below code, shows how to set Non-Retryable error and configure Activity retry options.

Custom Exception
```java
public class CustomNonRetryableException extends RuntimeException {
    public CustomNonRetryableException(String message) {
        super(message);
    }
}
```

### Activity
```java
@ActivityInterface
public interface SampleActivity {
    @ActivityMethod
    String performTask() throws CustomNonRetryableException, Exception;
}

public class SampleActivityImpl implements SampleActivity {
    @Override
    public String performTask() throws CustomNonRetryableException {
        // Example logic that may fail
        if (someCondition()) {
            throw new CustomNonRetryableException("This error is non-retryable");
        }
        return "Task completed";
    }

    private boolean someCondition() {
        // Implement your failure condition logic
        return false;
    }
}
```

### Workflow
```java
  @WorkflowInterface
  public interface SampleWorkflow {
    @WorkflowMethod
    String runSample();
  }

  public static class SampleWorkflowImpl implements SampleWorkflow {
    private final SampleActivities activities =
        Workflow.newActivityStub(
            SampleActivities.class,
            ActivityOptions.newBuilder()
                .setStartToCloseTimeout(Duration.ofSeconds(10))
                .setRetryOptions(
                    RetryOptions.newBuilder()
                        .setInitialInterval(Duration.ofSeconds(1))
                        .setDoNotRetry(CustomNonRetryableException.class.getName())
                        .build())
                .build());

    @Override
    public String runSample() {
      activities.performTask();
    }
  }
  ```

## Activity Timeouts and Heartbeats
We have already discussed the various timeouts in depth, however equally important, especially for longer running Activities are Heartbeats. Using a Heartbeat allows for Activity to let Temporal service know it is ok, and when it isn't, we can re-schedule Activities, without waiting the entire ScheduleToClose or StartToClose timeout. In addition, you can also return information in the Heartbeat details, which can be very helpful for ensuring an Activity continue where it left off, regardless of failures.

The below code, shows how we can process a file, periodically Heartbeating and returning the line we are on, so things can be continued. In addition, both the StartToClose and heartbeat timeout are set in our Activity options.

### Activity
```java
@ActivityInterface
public interface FileProcessingActivity {
    @ActivityMethod
    void processLargeFile(String fileName);
}

public class FileProcessingActivityImpl implements FileProcessingActivity {

    @Override
    public void processLargeFile(String fileName) {
        ActivityExecutionContext context = Activity.getExecutionContext();
        int totalLines = 100;  // Assume reading 100 lines for demonstration

        for (int line = 0; line < totalLines; line++) {
            // Do some processing here...

            // Heartbeat every 10 lines
            if (line % 10 == 0) {
                context.heartbeat(line);
            }

            try {
                // Simulate some processing time
                Thread.sleep(100);
            } catch (InterruptedException e) {
                throw new RuntimeException("Activity interrupted", e);
            }
        }
    }
}
```

### Workflow
```java
@WorkflowInterface
public interface FileProcessingWorkflow {
    @WorkflowMethod
    void processFile(String fileName);
}

public class FileProcessingWorkflowImpl implements FileProcessingWorkflow {

    private final FileProcessingActivity activities;

    public FileProcessingWorkflowImpl() {
        this.activities = Workflow.newActivityStub(FileProcessingActivity.class,
                ActivityOptions.newBuilder()
                        .setTaskQueue("file-processing-task-queue")
                        .setStartToCloseTimeout(Duration.ofMinutes(10))
                        .setHeartbeatTimeout(Duration.ofSeconds(30))
                        .build());
    }

    @Override
    public void processFile(String fileName) {
        activities.processLargeFile(fileName);
    }
}
```

## Versioning
Due to the requirement, that Workflows be deterministic, Workflow versioning, and choosing the right versioning strategy, are critical. Temporal offers several versioning strategies:
- Workflow Definition
- TaskQueue
- Patch

### Workflow Definition
Versioning strategy that involves versioning the Workflow definition file (MyWorkflowV1, MyWorkflowV2, etc). It involves creating new set of Workers for every version and updating both the Workflow starter as well as Worker. The V2 Workers handle V2 Workflows and once all V1 Workflows are complete they can be shutdown. This versioning strategy works best for short-lived workflows that don't have a lot of changes or versions.

### TaskQueue
Versioning strategy that involves versioning Workflow TaskQueue (MyTaskQueueV1, MyTaskQueueV2, etc). It involves creating new set of Workers for every version and updating both the Workflow starter as well as Worker. The V2 Workers handle V2 Workflows and once all V1 Workflows are complete they can be shutdown. This versioning strategy works best for short-lived Workflows that have a lot of changes or versions.

### Patch
Versioning strategy that involves versioning Workflow code and branching within Workflow code. Temporal SDK provides the ```Workflow.getVersion``` API which is used to set version and branch code. For example, below the ```Workflow.getVersionin``` API
sets the version to 1 in Workflow event history, and then is used to check if Workflow is the default version (our un-versioned or original Workflow code) and if it isn't then it will execute our new, updated Activity.

```java
int version = Workflow.getVersion("checksumAdded", Workflow.DEFAULT_VERSION, 1);
if (version == Workflow.DEFAULT_VERSION) {
    activities.upload(args.getTargetBucketName(), args.getTargetFilename(), processedName);
} else {
    long checksum = activities.calculateChecksum(processedName);
    activities.uploadWithChecksum(
        args.getTargetBucketName(), args.getTargetFilename(), processedName, checksum);
}
```

This versioning strategy works best for long-running Workflows, where the code branches can be maintained and removed as to not end up with an unmanageable amount of code branching.

## Cancellation and Compensation
Cancellation and Compensation are very important patterns in Temporal. They ensure failures are handled gracefully, while maintaining a consistent state. Cancellation allows for halting Workflow execution, while Compensation executes a predefined list of actions, in order to undo any changes that may have ocurred, during Workflow execution.

In Temporal, Compensation is just a simple try/catch, where upon error, any changes will be undone.
```java
try {
    order = activities.processOrder(orderId);
} catch (ActivityFailure e) {
    // Compensate Order (rollback)
    order = activities.compensateOrder(orderId);
    throw ApplicationFailure.newNonRetryableFailure(e.getMessage(), "OrderFailed");
}
```

Cancellation can be implemented using a CancellationScope. The scope can be attached or even detached from Workflow. When user (through API, CLI, UI) cancels a Workflow, that cancellation is passed into Workflow, and then acted upon (if a scope is implemented). This gives us a chance to perform any compensation and also important cleanup steps, before terminating a Workflow.

### Activity
```java
@ActivityInterface
public interface OrderActivities {
    @ActivityMethod
    void processOrder(String orderId);

    @ActivityMethod
    void compensateOrder(String orderId);
}

public static class OrderActivitiesImpl implements OrderActivities {

    @Override
    public void processOrder(String orderId) {
        // Simulate a long operation that can be canceled
        System.out.println("Processing order: " + orderId);
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            System.out.println("Order processing was interrupted.");
            throw e; // Rethrow to allow cancellation
        }
    }

    @Override
    public void compensateOrder(String orderId) {
        System.out.println("Compensating order: " + orderId);
    }
}
```

### Workflow
```java
@WorkflowInterface
public interface CancellationWorkflow {
    @WorkflowMethod
    String run();
}
public class CancellationWorkflowImpl implements CancellationWorkflow {
    // If cancellation received we should not wait for activity but try to cancel it
    private final OrderActivities activities = Workflow.newActivityStub(
            OrderActivities.class,
            ActivityOptions.newBuilder()
                    .setStartToCloseTimeout(Duration.ofMinutes(5))
                    .setCancellationType(ActivityCancellationType.TRY_CANCEL)
                    .build()
    );

    @Override
    public void run(String orderId) {
        CancellationScope scope = Workflow.newCancellationScope(() -> {
            try {
                activities.processOrder(orderId);
            } catch (ActivityFailure e) {
                // Check if the failure is due to cancellation
                System.out.println("Activity was canceled: " + e.getMessage());
                throw e;
            }
        });

        scope.run();

        // If the scope is canceled, run the compensation activity
        if (scope.isCancelled()) {
            activities.compensateOrder(orderId);
        }
    }
}
```

## Summary
In this article we discussed some of the common Temporal Workflow patterns. These patterns provide reusable solutions to common problems, encountered when building Temporal Workflows. Finally, we provided code samples, illustrating the various Workflow pattern implementations in detail. 

(c) 2024 Keith Tenzer




