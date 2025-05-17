--- 
layout: single
title:  "Agentic AI Frameworks"
categories:
- AI
tags:
- AI
- Agentic
- LLM
- Workflows
- Framework
- Temporal
- CrewAI
- LangGraph
- LangChain
---

![Agent](/assets/2025-05-16/agent.png)

## Overview
The future of applications is Agentic and that future is here, now. AI is transforming how we build and interact with software. Large Language Models (LLMs) are being integrated into applications to increase capabilities and enable autonomous task execution. It's no longer just about question-and-answer systems.

Agentic AI combines an LLM with tools—external applications, services, data sources, and more. At its core, an Agentic system is a framework that connects an LLM to a business process, effectively making it a workflow.

An AI Agent must handle the following:
- Managing conversations between the LLM and user
- Configuring the LLM with appropriate roles (system, user, assistant, tool), plans, and multi-shot examples
- Executing tools (e.g., API calls to services, databases, other LLMs, agents, RAG, inference, etc.)
- Storing conversation history and managing prompting

That’s a substantial amount of responsibility.

## Technology Stack
When selecting a technology stack for an AI Agent, some decisions are more straightforward than others. Here are key considerations:

- Do you plan to use a single frontier model provider and are comfortable with the associated vendor lock-in?
- Does your agent require durability? Is the use case mission-critical?
- Is the agent’s workflow complex, involving multiple steps, branches, or business logic?
- What level of scalability is required?
- Will the workflow involve interactive steps, such as user approvals?
- Does the agent require long-running conversations over extended periods?
- How much data will be passed into the LLM in each conversation turn?
- Do you require token-level streaming between the LLM and the user, or can responses be delivered all at once?

Below is a comparison of leading Agentic AI frameworks, highlighting their strengths and limitations.

| Framework     | Pros                                                        | Cons                                         |
| ------------- | ----------------------------------------------------------- | -------------------------------------------- |
| **Temporal**  | Durable, fault-tolerant workflows                           | Steeper learning curve                       |
|               | Polyglot SDKs (Go, Java, Python, .NET)                      | Operational overhead                         |
|               | Scales to thousands of concurrent workflows                 | May be overkill for simple agents            |
|               | Supports user interaction within workflows                  | Limited payload size & streaming support     |
|               | Rich observability (history view, metrics, tracing)         |                                              |
| **LangGraph** | Graph-based orchestration model                             | Early-stage project with a smaller community |
|               | Tight integration with LLMs like ChatGPT and Claude         | Limited built-in durability                  |
|               | Lightweight: no infrastructure required                     | Minimal UI and monitoring tools              |
|               | Extensible nodes and tool integrations                      |                                              |
| **CrewAI**    | Fully managed, low-code platform with drag-and-drop design  | Lack of state management                     |
|               | Hosted infrastructure (zero ops)                            | Less flexible, static workflows              |
|               | Built-in analytics and monitoring                           | Limited control over execution               |
|               | Enterprise features (RBAC, audit logs, SLAs)                | Lacks built-in durability and retries        |
|               |                                                             | No support for updates or feedback loops     |
| **OpenAI**    | Native integration with OpenAI LLMs                         | Tied to OpenAI ecosystem and pricing         |
|               | Simplified, code-first agent development                    | No built-in durability or retry mechanism    |
|               | Automated tool planning and execution                       | Limited observability and monitoring         |
|               | Low operational overhead                                    | Less flexible with non-OpenAI models         |

## Agentic Airline Assistant
I built an [Airline AI Agent](https://github.com/ktenzer/airline-ai-agent) using Temporal, LangGraph, and CrewAI to evaluate each framework’s strengths and weaknesses.

This AI Airline Agent specializes in searching and booking flights from Los Angeles to select destinations. It handles conversations using an LLM—configured with GPT-4o-mini, though any LLM can be used. It also calls tools such as `find_flights` and `book_flight`. All examples use a simple ChatUI built with [Gradio](https://www.gradio.app/) and [LangChain](https://www.langchain.com/) for LLM interaction, configuration, and prompting.

These examples allow you to explore each framework’s capabilities and trade-offs. I’ve aimed to remain as objective as possible.

## Recommendations
Based on my experience working with these Agentic frameworks, here are some practical recommendations:

- **OpenAI Agent SDK**: Best if you're using a single frontier LLM and are comfortable with vendor lock-in.
- **Temporal**: Ideal for any agent, especially if they require durability, fault-tolerance, scalability, and production-grade features. Also supports polyglot development.
- **CrewAI**: A good option for simpler agents where durability isn't critical and operational overhead must be minimal.
- **LangGraph**: Suitable for lightweight, code-first agents with tight LLM integration and limited durability needs.

## Summary
This article explored Agentic AI Agents, their requirements, and opportunities. We reviewed leading frameworks and compared their strengths and limitations. A simple AI Airline Assistant example demonstrated how each framework can be applied. Finally, practical recommendations were provided to guide framework selection.

(c) 2025 Keith Tenzer




