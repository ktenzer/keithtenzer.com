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

**Admission:** Update request is admitted as long as no [limits](https://docs.temporal.io/kb/temporal-platform-limits-sheet) are breached. 

**Validation:** Optionally Update input arguments can be validated and accepted or rejected.

**Execution:** Update is executed and delivered to the worker Update handler, WorkflowExecutionUpdateAcceptedEvent added to event history.

**Completion:** Update handler result returned from worker, WorkflowExecutionUpdateCompletedEvent added to event history.

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
As mentioned, Update really simplifies interactions between workflows or applications and workflows. Prior to Update, we would have to Signal a workflow to mutate it's state and then Query that workflow using polling or possibly Signal back after the mutation has completed. Polling with a Query isn't efficient and Signaling does not provide a response or any kind of validation. Now with Update all of these problems are solved, so lets take look.

The below illustration shows workflow interactions for a trivia game application built on Temporal. Players are added to the game workflow through the player workflow. In order to add a player the following steps are required:
- Moderate to ensure player name is not flagged through profanity filters.
- Query to check player is unique and not already added to game.
- Signal to add the player to the game.
- Query to check to ensure player is added.

![Trivia w/Signal and Query](/assets/2023-11-20/trivia-game-signal_query.png)

As you can see this involves two queries and a signal. It also requires polling from the second query, as we don't know how long it will take to add a player (mutate state). Alternately this could have also been implemented using two Signals, which would've removed need to poll, however also introduced other complexities. Neither solution is ideal.

Now let's look at this same diagram using the new Update primitive.

![Trivia w/Update](/assets/2023-11-20/trivia-game-update.png)

## Implementing Update
To implement Update we need to create an Update handler for receiving the update and optionally validating Update arguments. We also need to send the Update using the client. The game workflow will implement the Update handler and the player workflow will send the Update.

### Implementing Update Handle
For the game workflow, we need to validate new players being added to the game to ensure they are unique and don't already exist. If validation succeeds we will add the player to the players map which is maintaining state of players in the game.

Note: we are implementing validation using an anonymous Go func. The reason is so that the validation function has access to the players map which is set outside of the handler.

```go
// Setup update handler to perform player validation and add player to game state machine
func updatePlayer(ctx workflow.Context, players map[string]resources.Player) error {
	err := workflow.SetUpdateHandlerWithOptions(
		ctx,
		"AddPlayer",
		func(ctx workflow.Context, player string) error {
			score := resources.Player{
				Score: 0,
			}

			players[player] = score
			return nil
		},
		workflow.UpdateHandlerOptions{Validator: func(player string) error {
			log := workflow.GetLogger(ctx)

			if _, ok := players[player]; ok {
				log.Debug("Rejecting player, already exists", "Player", player)
				return errors.New("Player " + player + " already exists")
			}

			log.Debug("Adding player update accepted", "Player", player)

			return nil
		},
		},
	)

	if err != nil {
		return err
	}

	return nil
}
```

### Implementing Update Call from Client
Once our Update handler exists we can send the game workflow an update. This will happen from the player workflow.

Note: Update cannot yet be sent directly from workflow to workflow in workflow code so we are doing so using an Activity.

```go
func AddPlayerActivity(ctx context.Context, activityInput resources.AddPlayerActivityInput) error {
	logger := activity.GetLogger(ctx)
	logger.Info("GetRandomCategoryActivity")

	c, err := client.Dial(resources.GetClientOptions("workflow"))
	if err != nil {
		return err
	}
	defer c.Close()

	updateHandle, err := c.UpdateWorkflow(context.Background(), activityInput.WorkflowId, "", "AddPlayer", activityInput.Player)
	if err != nil {
		return err
	}
	var updateResult bool
	err = updateHandle.Get(context.Background(), &updateResult)
	if err != nil {
		return err
	}

	return nil

}
```

The entire pull request for switching to Update can be viewed [here](https://github.com/ktenzer/temporal-trivia/commit/9c4ab8bd0d83ee590c94898051689bb13a4be458).

## Summary
In this article we discussed the new Temporal primitive Update. We talked about how Update works and also provided a real-world example of switching to Update from Signal/Query. Using Update, we were able to validate a player being unique as part of the same call instead of separately. In addition we reduced Temporal primitive calls from three to just one. Finally, we were able to remove inefficient polling. I am excited to see what new Temporal patterns and use cases the Update primitive unlocks. 

(c) 2023 Keith Tenzer




