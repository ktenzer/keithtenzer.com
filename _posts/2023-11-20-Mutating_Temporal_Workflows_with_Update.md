--- 
layout: single
title:  "Mutating Temporal Workflows with Update"
categories:
- Temporal
tags:
- Workflow
- Update
- Mutating
- State
- Temporal Cloud
- Go
---

![Temporal](/assets/2022-08-15/logo-temporal-with-copy.svg)
## Overview
Temporal has just released a brand new primitive into it's programming model called [Update](https://docs.temporal.io/workflows#update)! As of the writing of this article, Update is in private preview in Temporal cloud. The new Update primitive enables developers to easily update workflow state while also validating those updates.

Update is a real game changer as it unlocks new use cases and possibilities for Temporal applications.
- Update replaces Signal/Response/Query pattern complexity with something easy to use and understand.
- Update is much more efficient than Signal/Query and does away with having to poll workflows.
- Update adds critical validation enabling accept/reject of Update parameters.
- Update better enables UIs or other such interfaces to be built on top of Temporal workflows.

## How Update Works?
Temporal Update has four phases.

**Admission** Update request is admitted as long as no [limits](https://docs.temporal.io/kb/temporal-platform-limits-sheet) are breached. 
**Validation** Optionally update input arguments can be validated and accepted or rejected.
**Execution** Update is executed and delivered to the worker update handler, WorkflowExecutionUpdateAcceptedEvent added to event history.
**Completion** Update handler result returned from worker, WorkflowExecutionUpdateCompletedEvent added to event history.

Update is currently a synchronous call, however it is planned to have an asynchronous version of the API in the future. In addition, the only way to call Update between workflows is to use an activity. It is planned to provide an API to call updates between workflows, similar to how Signal works today.

The below diagram illustrates how Update interacts between workflows, across the four stages (today being called from activity).

![Update](/assets/2023-11-20/update.png)

## Enabling Update
If you are using the Temporal development environment it can be enabled while starting the server.
``` bash
$ temporal server start-dev --dynamic-config-value frontend.enableUpdateWorkflowExecution=true
```

If you are using Temporal cloud, contact the sales team.

## Replacing Signal/Query with Update
As mentioned, Update really simplifies interactions between workflows or applications and workflows. Prior to update, we would have to Signal a workflow to mutate it's state and then Query that workflow using polling or possibly Signal back after the mutation has completed. Polling with a Query isn't efficient and Signaling does not provide a response or any kind of validation. Now with update all of these problems are solved, so lets take look.

The below illustration shows workflow interactions for a trivia game application built on Temporal. Players are added to the game workflow, through the player workflow. In order to add a player the following steps are required:
- Moderate to ensure player name is not flagged through profanity filters
- Query to check player is unique and not already added to game
- Signal to add the player to the game
- Query to check to ensure player is added

![Trivia w/Signal and Query](/assets/2023-11-20/trivia-game-signal_query.png)

As you can see this involves two queries and a signal. It also requires polling from the second query, as we don't know how long it will take to add a player (mutate state). Alternately this could have also been implemented using two Signals, which would've removed need to poll, however also introduced other complexities. Neither solution is ideal.

Now let's look at this same diagram using the new Update primitive.

![Trivia w/Update](/assets/2023-11-20/trivia-game-update.png)

Using Update, we are able to validate a player being unique as part of the same call instead of separately. In addition we reduced Temporal primitive calls from three to just one. FInally, we were able to remove inefficient polling.

### Query/Signal/Query Code (old)
The old code before Update.

```go
// Add player workflow definition
func AddPlayerWorkflow(ctx workflow.Context, workflowInput resources.AddPlayerWorkflowInput) error {
	logger := workflow.GetLogger(ctx)
	logger.Info("Trivia Game Started")

	// Activity options
	ctx = workflow.WithActivityOptions(ctx, setDefaultActivityOptions())
	laCtx := workflow.WithLocalActivityOptions(ctx, setDefaultLocalActivityOptions())

	activityInput := resources.QueryPlayerActivityInput{
		WorkflowId:      workflowInput.GameWorkflowId,
		Player:          workflowInput.Player,
		NumberOfPlayers: workflowInput.NumberOfPlayers,
		QueryType:       "normal",
	}

	// run activity to check for valid name
	moderationInput := resources.ModerationInput{
		Url:  os.Getenv("MODERATION_URL"),
		Name: workflowInput.Player,
	}

	// run moderation activity to check provided name
	var moderationResult bool
	modErr := workflow.ExecuteActivity(ctx, activities.ModerationActivity, moderationInput).Get(ctx, &moderationResult)
	if modErr != nil {
		logger.Error("Moderation Activity failed.", "Error", modErr)
		return modErr
	}

	if moderationResult {
		return errors.New("Player name is invalid")
	}

	// Check if player is unique or already exists
	var isPlayer bool
	err := workflow.ExecuteLocalActivity(laCtx, activities.QueryPlayerActivity, activityInput).Get(laCtx, &isPlayer)
	if err != nil {
		logger.Error("Query Player Activity failed.", "Error", err)
		return err
	}

	if isPlayer {
		return errors.New("Player " + workflowInput.Player + " already exists or game full!")
	}	

	// Add player via signal
	addPlayerSignal := PlayerSignal{
		Action: "Player",
		Player: workflowInput.Player,
	}

	err = workflow.SignalExternalWorkflow(ctx, workflowInput.GameWorkflowId, "", "add-player-signal", addPlayerSignal).Get(ctx, nil)
	if err != nil {
		return err
	}

	// Ensure player is added to game
	activityInput = resources.QueryPlayerActivityInput{
		WorkflowId: workflowInput.GameWorkflowId,
		Player:     workflowInput.Player,
		QueryType:  "poll",
	}

	err = workflow.ExecuteLocalActivity(laCtx, activities.QueryPlayerActivity, activityInput).Get(laCtx, &isPlayer)
	if err != nil {
		logger.Error("Query Player Activity failed before adding to game.", "Error", err)
		return err
	}

	if !isPlayer {
		return errors.New("Player " + workflowInput.Player + " could not be added to the game")
	}

	return nil
}
```

### Update Code (new)
The new code after Update.

```go
// Add player workflow definition
func AddPlayerWorkflow(ctx workflow.Context, workflowInput resources.AddPlayerWorkflowInput) error {
	logger := workflow.GetLogger(ctx)
	logger.Info("Trivia Game Started")

	// Activity options
	ctx = workflow.WithActivityOptions(ctx, setDefaultActivityOptions())
	laCtx := workflow.WithLocalActivityOptions(ctx, setDefaultLocalActivityOptions())

	activityAddPlayerInput := resources.AddPlayerActivityInput{
		WorkflowId: workflowInput.GameWorkflowId,
		Player:     workflowInput.Player,
	}

	// run activity to check player name against moderation api
	moderationInput := resources.ModerationInput{
		Url:  os.Getenv("MODERATION_URL"),
		Name: workflowInput.Player,
	}

	var moderationResult bool
	modErr := workflow.ExecuteActivity(ctx, activities.ModerationActivity, moderationInput).Get(ctx, &moderationResult)
	if modErr != nil {
		logger.Error("Moderation Activity failed.", "Error", modErr)
		return modErr
	}

	if moderationResult {
		return errors.New("Player name is invalid")
	}	

	// Call update through activity to add a player
	err := workflow.ExecuteLocalActivity(laCtx, activities.AddPlayerActivity, activityAddPlayerInput).Get(laCtx, nil)

	if err != nil {
		return errors.New(err.Error())
	}

	return nil
}
```
The entire pull request for switching to update can be viewed [here](https://github.com/ktenzer/temporal-trivia/commit/9c4ab8bd0d83ee590c94898051689bb13a4be458).

## Summary
In this article we discussed the new Temporal primitive Update. We talked about how Update works and also provided a real-world example of switching to Update from Signal/Query. I am excited to see what new Temporal patterns and use cases the Update primitive unlocks. 

(c) 2023 Keith Tenzer




