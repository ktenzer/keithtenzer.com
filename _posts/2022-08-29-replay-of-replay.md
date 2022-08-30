--- 
layout: single
title:  "Replay of Replay"
categories:
- Temporal
tags:
- Temporal
- Opensource
- Workflows
- Activities
- Go
- Replay
---

![Replay](/assets/2022-08-29/temporal-replay.png)
# Overview
Temporal just completed it's inaugural user conference called Replay in Seattle. As such I wanted to do a quick replay of Replay. First, why Dinosaurs? Well, Dinosaurs are cool right? If Dinosaurs were a Temporal application we could replay them, experiencing the past over and over, now wouldn't that be cool!

# Impressions
Honestly, replay blew my mind. I've been to a lot of conferences but this was the first time I felt I could relate to everyone. That we all not only shared similar challenges, but also a vision for the future. It was collaborative, informative and truly visionary. The community that came together represented some of the most iconic companies on our planet, disruptive startups and large global enterprises. All with incredible challenges to solve and looking at Temporal for at least part of the solution. 
Scaling applications at the cost of complexity or increasing developer velocity at the cost of reliability and safety, simply are not tradeoffs we are willing to accept. There is a better way and this community seems on the cusp of re-inventing or even disrupting an entire industry. How we build, connect, deploy applications and the platform services they depend on.

# Day 1
The first day had split tracks with birds of the feather discussions in the lounge area and workshops in the main hall. I attended the workshop sessions which had a great mix of hands-on, real-world customer examples and theoretical.

### Temporal Service and Application Architecture
This was a workshop led by several Temporalers (including one it's co-founders Samar Abbas) to dive into real customer applications and answer their questions. Samar began by eloquently explaining how using Temporal really forces you to think differently. Most (including myself) think about a workflow as having a start and an end but Temporal challenges that by removing constraints of time. In Temporal, a workflow encapsulates processes and entities, reacting to signals sent and as such can run infinitely. Temporal only executes a process when it has something to do. The only real limitation of a workflow is 50k events or 50Mb of storage in the event history. These limitations however, have a simple solution, using [continue-as-new](https://docs.temporal.io/concepts/what-is-continue-as-new/) we can simply refresh the event history for a given workflow. 

In this workshop we reviewed code and some of the challenges from two Temporal users: Tock and Coalition.

#### Tock
Provides a reservation system used by many restaurants. They are using Temporal for the following use cases:  Determining reservations offered by restaurants, sending emails via typescript, e-commerce flows and scheduled workflows. 

###### What is best practice when we don't want a retry policy?
Temporal will change how you think about failure and will motivate you to think about retrying until it is repaired. Don't fail a workflow, instead fix what is preventing workflow from proceeding.

###### How to keep long running workflows without if/then/else logic to handle versioning?
Provide versioning primitive to batch long runnings apps. Use continue-as-new which restarts a workflow with same input, just clearing it's history. Have guarantee that no workflows rely on a version older than X of the client. If you do have really long running workflows and require introducing a new worker, consider keeping old worker around till those workflows complete. New workflows can then execute on a new worker.

###### What is best way to share output types from workflows written in different languages?
Everything in temporal is serialized, can use data converters and write your own or protobuff wrappers at gRPC level.

###### How to prioritize different types of workflows?
Higher priority workflows should have dedicated task queue with dedicated pool of workers. Works well if you can create pools that have a few different priority workflow types and not large numbers of priorities. Can also be done at workflow level, each signal is an order to execute a child workflow and business logic can determine priority in this case.

###### How do you access longer retentions (3-6 months)?
All workflow state is kept up to 30 days after a workflow closes in the event history. Some want to keep this around for a longer, due to compliance issues. Recommendation is to use archive feature only currently available in the OS. Archive empties the state in Temporal's backend and pushes it to a blob.

###### General recommended strategy for scaling workers? If sharing same resources, does it make sense to have separate workers? What about serverless approach to workers?
Not opinionated on how to deploy workers, workers are stateless, they maintain a cache for performance reasons. You can create dynamic autoscale approaches to workers. Other piece is isolation which is at Temporal namespace level. Temporal has strong guarantee for availability and performance of a given namespace. For example limiting blast radius or noisy neighbor scenario. Example, a bug in a worker for a given workflow could cause OOM issues and that could propagate through workers affecting other workflows that may be of higher priority.

###### How to find backlog of task queue to understand how to schedule workers? 
Temporal does not yet allow directly controlling task queues but the Schedule-To-Start timeout allows you to limit backlog on task queues by re-routing or draining them to a different task queue / worker.

#### Coalition
Focuses on active side of cyber security insurance. Helping customers avoid security breaches. They are using Temporal for the following use cases: lifecycle of policy, sending emails and anything in general that comes from a request should go through Temporal.

###### When does it make sense to do activity vs job workflow?
Everything that does not have to do with orchestration should be done via Activity. For example, using timers, business logic or integrating with external systems should all be done within activities. Job workflow can be used in order to partition some limitations of histories for long running workflows.

###### When to use new workflow vs child workflow?
Temporal scales on number of workflow executions but an individual workflow execution has limits as already explained. Child workflows can be used to partition a workflow as each child workflow has it's own event history. A child workflow execution should maintain a one to one resource mapping and be considered a separate service.

###### Where to use Sleep?
Sleep is one of blocking APIs to create a durable timer which is duration of sleep. Polling in workflow however can blowup history of workflow execution. A way around this is to use signals. For example, might have external system polling for events in Temporal and sending signal to workflow when polling completes to continue with next activity.

###### How much data should I send to Temporal?
Metadata is generated internally but the majority of overall data within Temporal is from input parameters and result of activity functions. All those payloads get serialized and stored in the history. One shouldn't pass large amounts of data, i.e. megabytes. Better to pass pointers to a blob that contains the actual data.

### Temporal 101 Workshop
This was a two part hands-on workshop intended for new users of Temporal (like me). An entire development environment was provided via GitPod. The exercises involved bootstrapping a Temporal worker, creating some basic workflows that consume a service and using UI/CLI utilities. Everything is available [here](https://learn.temporal.io/replay2022) and I highly recommend this for anyone new to Temporal. In addition if you want to setup your own local development environment, I provided a consolidated guide [here](https://keithtenzer.com/temporal/temporal_getting_started_guide/).

### Building event-driven, reactive applications
This session was very thought provoking. It started with discussing around processes; which essentially do two things: coordination and execution. In Temporal coordination is the workflow and execution is the activity. Generally coordination is very long lived while execution tends to be short lived. There are two very different notions for how to achieve long running coordination: orchestration vs choreography.

#### Orchestration
Command based (step 1, step 2, step 3), explicit control flow and direct. Behavior is pre-determined

#### Choreography
Does not depict behavior, event based, implicit flow control and indirect. Behavior comes to life after a process executes.

While both notions work, providing solution to long running coordination; only orchestration does it simply with much less complexity. Choreographed systems require event buses, streams, retry logic and many databases to store persistency at every change. Behavior as such is inherently difficult to understand and when things go wrong, challenging to troubleshoot. Temporal is an orchestrated system, solving the problem with less code and complexity, resulting in higher reliability and simplicity. 

# Day 2
The second day featured keynotes and Temporal user sessions from some quite amazing speakers.

### Keynote: Maxim Fateev
Maxim is Temporal's CEO and co-founder. He provided not only a brilliant history of temporal but also compared building applications to fruits. Often we start with a fruit (microservice) but then it needs to interact with other fruits, since we are of course mixing fruits, to make an amazing desert (application). The result is a smoothie. While it can taste great, it is impossible to actually know what ingredients are in there without reading a step-by-step of how it was created. The smoothie approach doesn't work well for applications as it is complicated to understand and replay. Reliability and durability are much lower as a result. Maxim explains a better way is to make a cake where each fruit is a layer. Contracts and boundaries are easily understood. If you want banana and strawberry but not blueberry then you just get a piece of those layers. This is the simplicity we can have with Temporal.

Mike Nichols did a fantastic live demo showcasing a sneak peak at the upcoming Temporal namespace-as-a-service cloud offering. The Temporal cloud not only provides a fully supported managed namespace but also helps get customers going quickly. Few users really want to standup and be responsible for the infrastructure running the Temporal server or namespace. It is easy to get something working (day 1) but building a scalable, reliable operating model (day 2) is a whole other story. While Temporal has been in production with a cloud service for a while, we have not yet offered namespace-as-a-service, namespaces were provided via support tickets. Soon, anyone will be able to enjoy the Temporal cloud and self provision their namespaces. 

### DataDog: Jeremy LeGrone
At DataDog the main uses of Temporal are for deployments, infra, control planes, datastore operators, code merges, queuing systems and incident response. 

#### Getting started recommendations
1. Define payload types as protobuf messages
2. Stick with single SDK language
3. Consider one of new SDKs like typescript 
4. Safeguard against making network or other blocking calls inside workflow
5. Don't allow Temporal API to be exposed to public, does not have protection against DDOS attack
6. Invest in core Temporal enablement team

Two challenges that DataDog encountered were around workflow versioning and large payloads.

#### Workflow versioning
Proposal in place using build IDs to provide solution. It would enable A/B deployments for example, allowing old v1 workflows to run on the v1 workers while new workflows go to v2 workers. In meantime until feature is implemented, you can do CI checks that do replay tests or use continue-as-new to allow long running workflows to restart on new v2 workers.

#### Large payloads
Storage is over the entire history of workflow, not just the current execution. Large payloads should be kept in blob storage space and you should pass only pointers as inputs. Downside is activity code needs special treatment and overall it can be complex to deal with lifecycle management.

### Square: Ryan Walls
At Square the main use cases for Temporal are service orchestration as code, better mechanism for retries, cancellations and increased durability for massive amounts of microservices.

Ryan focused his session about taking innovation from idea to broad enterprise adoption. It was very relevant, as many developers using Temporal today or who are just learning about it need to grow adoption within their organizations. That only happens if you can influence and be a catalyst for change. Instead of winging it like most of us, Ryan shared his learnings from taking idea to adoption in very large innovative organizations. He shared great insights and also captured some of the pitfalls. 

### Stripe: Drew Hoskins
Drew stated that Temporal is a paradigm shift for distributed systems. I think one of the key takeaways from replay was that we as a community are at the beginning of something big that will re-shape how we think about and build future applications.
At Stripe, Temporal is used by both infrastructure and product teams. The priorities at Stripe are developer productivity and reliability. It isn't good enough to move fast, they have to do so safely and reliably. 
Client teams own Temporal workers and monitoring. Platform at Stripe offers fully wrapped Ruby / Java SDKs, lightly extended GO SDK for infra team, Temporal server / UI, proxy and management plane.

#### Why wrap the SDK?
Stripe wanted to build sorbet types (strong typing), actionable error messages and inspect URLs (links to Temporal docs) which explains how to debug certain known issues that Stripe developers typically encounter. It also lets you guide developers better to make the correct implementation choices,keeping certain guardrails around them.

#### Art of timeouts 
1. A workflow has been stuck for hours and nobody knows why?
   - Its waiting on activity which has long timeout. Either set shorter timeout or heartbeat
2. Workflow failed because activity took too long to start
   - Never use schedule-to-start timeout, monitor instead
3. Workflow timeout after 30 mins due to a bug (use tctl to fix)
4. Use long workflow timeouts, stripe defaults to 3 weeks so there is time to deploy fixes
5. Discourage start-to-close timeout of 2 hours or more

#### Art of retries
1. Workflow activity that moves money, succeeds but failed to report due to network issue could result in money moving twice
  - Make all activities idempotent
2. Cluster goes down and need to fail running workflows over to another region
   - Ensure all activities are idempotent allowing them to be re-run safely
3. Have test framework that tests for idempotency 
   - Once activity goes into integration, test behind the scenes for idempotency

#### Workflow latency SLAs
1. A workflow encounters transient failure
   - Track failure metrics and alert if too high
2. Workflows encounter extended outage
   - Same but notify developer
3. Code bug, invalid argument
   - Same developer deploys fix and workflow resumes
4. Have ton of workflows that are stuck
   - Add OutOfTimeSla search to elastic search, teams can monitor with a recurring check

#### Batches
Workflows that perform an operation on set of items (batched for efficiency)

1. Batch activity takes long. Still running?
   - Use activity heartbeats
2. In failure loop, every time it fails, have to start activity from beginning
   - Pull out cursor and pass to heartbeat so you can resume where you left off
3. Processing huge number of records in parallel across many activities, workflow history gets too long
   - Use continue-as-new or child workflows diving work among multiple workers


### Keynote: Paul Nordstrom
Paul took us on a wonderful journey which compared application ecosystems to our own biological ecosystems. It paralleled how things evolved in both software and the biological world. Paul shared a powerful vision based on observing biological worlds, what is the logical next step for software and applications? 
Paul theorized that the next step in software evolution would be application or services communities. The biological world depends on contracts and a certain balance between all life organisms within the ecosystem. Similarly applications too should develop well-defined contracts in how they interact with themselves and other critical dependencies. Something like Temporal could evolve to provide a way for applications or services to create these contracts which enforce durability and reliability, allowing interactions to be quite predictable.

### Yum! Brands: Matt McDole
After hearing from Temporal's CEO, some of the most iconic voices in our industry and a peak through a visionary's lens, it was time to hear a story from a large scale enterprise user. I felt like we saved the best for last. Matt delivered an extremely insightful, thoughtful presentation on not only how but why Yum! brands is using Temporal.
Using Temporal Yum! brands achieved: simplified CD, increased oder flow reliability for KFC (Taco Bell/Pizza Hut are next), improved idempotency and simplified production support. Just using Temporal alone removed an estimated quarter of their code base. They no longer needed to maintain their own pollers, task queues, retry logic or state machine. The result was an increased overall reliability and serviceability. Matt gave us a real world understanding of why you never want to be debugging a state machine during a production down event.
What I learned from Matt was how easy it is to start with Temporal. You don't need to boil the ocean; instead you can start small, for example replace retry logic of a single service and then build form there. With each successful use case the value of Temporal grows. Using the Temporal cloud is a very fast, low cost way to get started. You don't need to stand-up a Kubernetes cluster, databases or even an SRE. If something isn't easy or cost effective it won't get broad adoption no matter what.

### Pannel: Shaping future of backend development
The replay conference ended with a panel of experts. Folks from Temporal, Snap and Netflix all represented the panel. They discussed what they would like to see in the future of backend development. A lot of the ideas discussed revolved around two kep points: moving fast (high development velocity, which we achieve today) but doing it safely (which is the missing piece). 

## Summary
In this article we replayed replay, Temporal's inaugural user conference. The first one of (I hope) many such conferences. My main takeaway is that we are at the beginning of a journey, launch of a new better way to build applications for durability. We don't even yet know how to call our new industry of durable applications but we will figure it out together, like everything else one step at a time! Best of luck in your journey to more reliable applications with Temporal! If you have any questions reach out on [Temporal community slack](https://temporal.io/slack).

(c) 2022 Keith Tenzer




