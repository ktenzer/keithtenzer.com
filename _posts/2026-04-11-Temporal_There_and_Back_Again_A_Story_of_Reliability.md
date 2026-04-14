--- 
layout: single
title:  "Temporal: There and Back Again - A Story of Reliability"
categories:
- Temporal
tags:
- Temporal
- Reliability
- AI
- Culture
- Durable Execution
---

![Temporal](/assets/2022-08-15/logo-temporal-with-copy.svg)

## Reflections
I came back to Temporal after what I am referring to as an "AI Sabbatical". In this blog, I am going to talk about that experience, why I left, and more importantly why I came back. Coming back for 2.0 vs [My First Day at Temporal](https://keithtenzer.com/temporal/my-first-day-at-temporal/) is a completely different experience, as you can imagine. I am different, Temporal is different, expectations are different, but one thing is the same. It's the core reason I came back, the reason I left an AI startup that had just grown well over 500% YoY.

The energy in AI startups is real: the sense of purpose, changing the world, being part of something that will change everything. This is what drove me to leave amazing colleagues, customers, and a company I truly believed in. It was not an easy decision. On the surface it seems like a rather irrational decision, especially considering I came back. Something you might do in a mid-life crisis, and maybe it was! Honestly, if I could go back in time knowing what I know now, I would have never left. I know, the classic hindsight is 20/20, but the key is "knowing what I know, which I didn't know then." I went on this journey because I needed to know. Always follow your heart. Life is too short to spend a day doing something you don't want to be doing or something that doesn't bring you purpose. All the money in the world won't bring happiness, it's just a convenience.

This journey has taught me patience, humility, honor, and respect, and I am so grateful for it. Rarely in life do we get a second chance. Often we deserve one, we want one, but it isn't entirely in our control. An opportunity presented itself for 2.0 and I didn't know what to expect from my ex-colleagues, or the ones who had joined after I left Temporal. Would I be welcomed back? Would I be able to reconnect, reinvent? Nothing is certain, but I took the chance anyway because of a core belief.

## An Opportunity
After I accepted the offer, I was greeted by 20+ LinkedIn messages. My return was even announced at an all-hands. I can't describe how humbling that was, and how good it made me feel to be appreciated in that way. I wasn't anyone special at Temporal to deserve that kind of welcome either. In fact, one thing that attracted me to Temporal in 1.0 was "No Titles." Yes, in the early days everyone was just engineer, or in my case, solution architect.

Temporal has always been about the people: employees, customers, shareholders, and something most companies leave out, family. People make life worth living and people are what make a great and lasting company. There are many great people at Temporal and I could write for days about all of them. One person, however, defined what this company is all about: Joe Miklos. He started as the first AE (Customer Account Manager) and I had the honor of being his SA (Solution Architect). He said something to me once that defines what Temporal is all about, something I will take with me to my grave and tell everyone about on the way: "You just got to show up for people." It is so simple, so true, and it took me this entire journey to really comprehend it.

## The Boomerang
The core reason I am back at Temporal is because I believe in something, I want to believe in something: Reliability. The definition of reliability is "showing up" for ourselves, our loved ones, our friends, family, colleagues, and customers. This is what Temporal is all about and this gives me true purpose. It's in everything we do. We sell Reliability in a world where everyone just wants to go fast. All the headlines are grabbed by speed. Yes, we need to go fast and it's easy to do. You certainly get a lot more headlines and hype for showing off how fast you can go, but nobody rewards you for crashing into a wall or being reckless. AI gave us rocket fuel, but if we don't control it, if we don't have guardrails, we will just burn up more quickly. The challenge today isn't going fast, it's going fast reliably. Temporal is key to ensuring we go fast safely, securely, and reliably.

Samar, our CEO, in the early days often talked about "Reliable as running water." In a world that keeps speeding up, you can sell more speed or be a contrarian and sell the reliability that enables it. Reliability quietly keeps your organization from ending up in a shipwreck at the bottom of the ocean. There are sadly many, many of those ships, and ones that are sinking as I write. You see, reliability doesn't just impact the product you are selling, your brand reputation, or value proposition. It affects the people who work there and their families. They burn out, get tired of plugging the dam with their fingers because systems weren't designed with reliability in mind. Eventually enough holes sink the ship, although a quicker end by spectacularly blowing up leaving the launch pad is also possible. Most AI companies today will sink or explode (I would say 90%), and the reason: most of them have not made reliability a focus.

## Temporal Value Proposition
Temporal is built for exactly the gap the AI era created: you can move quickly while your application and platform maintain reliability. Temporal can even go back in time without losing your place in the future (it's called Replay)!

**Durable execution** means workflow state survives process restarts, deploys, and transient failures. An AI agent or a human approval step can sit in the middle of a process for minutes or months without you re-implementing checkpoints and idempotency by hand every time.

**Observability** is first-class: history, visibility APIs, and operational tooling that let you answer what happened, what is in flight, and what to replay or reset, without guessing from scattered logs.

**Scalability** comes from the worker model and the separation of orchestration from execution, so you can add capacity where the work is without rewriting the story of the workflow.

Together, those capabilities are the guardrail for the AI-driven economy. AI agents can iterate on steps governed by durable workflows, timeouts, retries, and explicit failure handling, preventing mistakes from turning into outages.

## Recent Lapses in Reliability
For years we chased velocity. With AI in the loop, we finally got the throughput everyone said they wanted: more code, more content, more automation, less time in the loop. The trade many teams made, often without saying it out loud, was to borrow against reliability. Corners get cut, reviews thin out, and production becomes the test plan.

The incidents below are not all "AI failures" in a narrow sense. They are what happens when complex systems and fast release cadences meet weak guardrails: a small mistake or a bad query fans out globally.

### Claude Code npm release and source map exposure (March 2026)
On March 31, 2026, version 2.1.88 of the public `@anthropic-ai/claude-code` npm package reportedly included a large JavaScript source map (on the order of tens of megabytes) that exposed extensive internal TypeScript detail from the product. Coverage described rapid discovery and mirroring across the ecosystem within hours. Anthropic told VentureBeat the issue was a release packaging problem caused by human error, not a security breach in the sense of customer data or credential exposure, and that it was implementing preventive measures.

**Source:** [Claude Code's source code appears to have leaked: here's what we know](https://venturebeat.com/ai/claude-codes-source-code-appears-to-have-leaked-heres-what-we-know/) (VentureBeat, March 31, 2026)

### Cloudflare BYOIP BGP withdrawal (February 20, 2026)
Starting February 20, 2026 at 17:48 UTC, a subset of customers using Cloudflare's Bring Your Own IP (BYOIP) saw their routes withdrawn over BGP. Cloudflare's post-incident write-up attributes the cause to a change in how the network manages BYOIP addresses: a scheduled cleanup sub-task called the Addressing API with a faulty query (`pending_delete` passed with no value), which the server treated in a way that led to mass withdrawal of customer prefixes. Roughly 1,100 BYOIP prefixes were withdrawn before the task was stopped. Cloudflare states the full customer-visible recovery window lasted 6 hours and 7 minutes, with much of that time spent restoring prefix configuration. The company explicitly ruled out cyberattack and detailed remediation plans including circuit breakers, safer automation, and related hardening.

**Source:** [Cloudflare outage on February 20, 2026](https://blog.cloudflare.com/cloudflare-outage-february-20-2026/) (Cloudflare Blog, February 21, 2026)

### GitHub platform incidents (February 2026)
GitHub published a February 2026 availability report describing six separate incidents with degraded performance across core services. Examples from that month include: Dependabot degraded for over an hour due to failover hitting a read-only database; a roughly six-hour window where GitHub Actions hosted runners and Codespaces were unavailable, rooted in mistaken security policies applied to backend storage that blocked VM metadata access, with spillover to Copilot agent, CodeQL, Pages, and more; and two related multi-service degradations on February 9 (combined about two hours and forty minutes) traced to a user-settings cache change that triggered massive cache rewrites and cascaded into Git HTTPS proxy connection exhaustion. The report is explicit that growth, coupling, and configuration risk amplified blast radius.

**Source:** [GitHub availability report: February 2026](https://github.blog/news-insights/company-news/github-availability-report-february-2026/) (GitHub Blog, March 11, 2026). For root-cause themes and remediation direction, see also [Addressing GitHub's recent availability issues](https://github.blog/news-insights/company-news/addressing-githubs-recent-availability-issues-2/).

### Internet-wide outage risk in 2026 (industry view)
Cisco ThousandEyes frames early-2026 risk in terms of interdependence between providers and the way automation and operational tooling can create cascading failures that are hard to predict from any single component. That matches what telemetry vendors and postmortems keep showing: not only "something broke," but "something changed quickly and the graph amplified it."

**Source:** [Looking Ahead: 2026's Biggest Outage Risks](https://www.thousandeyes.com/blog/internet-report-2026-biggest-outage-risks) (ThousandEyes Blog)

The point is not to create cynicism about shipping. It is that speed without durability, observability, and clear blast-radius limits converts local mistakes into organizational wide outages that damages an organizations reputation and core-value proposition.

## A Final Thought
This post is a personal note about being a human, returning to Temporal, and doubling down on reliability. A question to ponder: if you could choose between features shipped or incidents avoided, how would you choose? The honest answer explains whether your culture rewards motion or reliability. I am expecting most would say motion, and I agree, as long as you have Temporal!

(c) 2026 Keith Tenzer
