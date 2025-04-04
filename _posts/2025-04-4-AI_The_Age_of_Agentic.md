--- 
layout: single
title:  "AI: The Age of Agentic"
categories:
- AI
tags:
- AI
- Agentic
- Temporal
- LLM
- MCP
---

![Temporal](/assets/2022-08-15/logo-temporal-with-copy.svg)
## Overview
When most people think about AI, they imagine robots doing work for of humans. A robot is just hardware, but its brain is software, more specifically an Agentic AI - acting autonomously, on behalf of a human, carrying out various tasks with limited intervention.

Agentic is a broad category within GenAI that can be broken into sub-catagories: interaction vs non-interaction. Interaction involves conversation with a human, while non-interaction can happen without a human even being aware. The key to both and what makes it Agentic is the use of GenAI coupled with acting on behalf of a human. 

## Agentic Requirements
Agentic AI requires new approaches and architecture. Message or event-driven architectures don't really fit well. In addition, new ways of connecting Agentic to data are emerging. One such standard from Anthropic, called Model Context Protocol [MCP](https://www.anthropic.com/news/model-context-protocol) has already started to gain traction. On one hand, Agentic requires focus on a conversation-driven approach, where the clear boundaries and definitions provided by a message or event-driven architecture, simply do not exist. On the other hand, it also couples business processes with GenAI, all of which need durable orchestration to be reliable. These are two distinct approaches and both are somewhat interwoven.

### Conversation-Driven Approach
Conversations are very verbose and happen in real-time. To handle a conversation of unlimited size over unknown duration, a layer between the human and LLM which can effectively stream tokens is required. In addition, the conversation must be persisted, so that it isn't lost or forgotten. 

### Non-Conservation-Driven Approach
An LLM is a generalist and will always require specialist tools to complete tasks. These tools could be other LLMs/SLMs, Inference, Retrieval-Augemented-Generation, even sub-agents. A tool will often be used in the background, without direction of a human. In addition, such tools will be coupled with business processes to achieve meaningful outcomes.

Lets consider a high-level architecture.
![Agentic architecture](/assets/2025-04-02/AI_Agentic_Architecture.png)

## Agentic with Temporal
Choosing the right technology to build Agentic upon is crucial. Orchestration is important, but only orchestration that can support long-running (infinite), while providing state management and durability. Temporal becomes a natural fit for orchestrating Human-to-AI delegation.

Temporal Advantages for Agentic
- Support Long-running  
- State Management
- Durability 
- Resource Throttling/Locking (GPUs)
- Scale
- Visibility
- Programming Model

As already discussed there are two aspects to Agentic: Conversation and Tool. Using Temporal, both can be modelled as Workflows.
![Agentic with Temporal](/assets/2025-04-02/AI_Agentic_with_Temporal.png)

### Conversation
The conversation Workflow will not only keep state (through any kind of failures), but also coordinate sending prompts to LLM, storing conversation text chunks and returning output tokens to the human. Pointers that map to the tokens are used, allowing for storing of the conversation state, directly within the Workflow. In addition, since Temporal Workflows support interaction through Signals or Updates, implementing human-approval for certain LLM related tasks is trivial in an Agentic Workflow.

### Tool
The tool Workflow represents a tool that can accomplish a specific task with a defined spec. Tools of course can be flaky or have low throttling, but Temporal will automatically handle retries. Temporal Workflows are code and can be written in Python using the Temporal SDK. The programming model provides an elegant abstraction for "durable execution" of tasks and enables developers to leverage any python AI frameworks or libraries, such as LangChain.

## Summary
Agentic AI is an incredibly exciting and rapidly growing space. We are probably only 12-18 months away from personal AI assistants which will help us carry out not only daily tasks, but improve our own capability, in private life and the workplace. Temporal is a great technology choice for building Agentic AI. It provides not only the orchestration capabilities for long-running, but also critical durability needed to ensure Agentic AI is reliable, and can be counted on just like its human counterparts.  

(c) 2025 Keith Tenzer




