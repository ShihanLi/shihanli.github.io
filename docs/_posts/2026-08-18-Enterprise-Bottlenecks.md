---
layout: post
title:  "Enterprise Bottlenecks"
date:   2026-08-18 23:10:43 -0800
categories: essays
---


If you are an engineer working at a big company, chances are you are not seeing the 10x productivity gains from adopting AI. 

You are busier than ever, consuming more tokens and generating more PRs. But you are not shipping features any faster. To meet leadership's expectation of a faster timeline, you feel the pressure to work longer hours to compensate. 

This is a common story I've heard from folks attending the [Boston QCon AI](https://boston.qcon.ai/) conference. And I learned from [Lizzie Matusov](https://boston.qcon.ai/keynote/boston2026/five-stages-ai-maturity-engineering-organizations-where-and-why-teams-get-stuck) that this discrepancy between expectation and reality is due to having bottlenecks. 

According to the theory of constraints, every system is limited by a single bottleneck. Speeding up anything else does not increase output, but rather piles work up in front of the slow step. Big companies are breeding grounds for bottlenecks due to the large number of processes they've accumulated. Some of them are bound to be slow. 

Here are a few I've noticed and our playbook in getting around them. 

Code reviews are a big one. The intuitive explanation is that more code being generated means more code needs to be reviewed. Since engineers have limited time, reviews start to pile up. While this makes sense, it's not what I've been seeing in my day to day. 

Within my team, speed of reviews has not been a problem. Sure there are more PRs to review, but the overall time reviews take up in our work is still very small. The slowest reviews, however, come from those where we need to get approvals from another team. 

In our organization, each team owns a few microservices so that we can develop features, scale, deploy, and maintain our own services independently from other teams. In reality, most features that we work on span across many services we don't own. So to ship a feature, we need to get alignment from other teams to approve our design, PRs, and finally our deployment requests. 

This process is slow because the teams who own the service often don't have the specific context needed to review the feature change. Regardless, they feel the need to review and understand the change because they're on the hook for servicing. If the SLA of their application dips, they have to pick up the incident call in all hours of the day to root cause. Almost always, they transfer the incident to our team once they realize we own the specific feature causing the alert. 

This process taxes this other team's time and energy twice. Once for having to understand things they don't need to understand, and another for being woken up for things they can't mitigate. 

It'd be better to invest in smarter incident routing that's not simply based off of who owns the service, but one that can find the context around the broken thing by looking at telemetry, code, history of change, and so on and route the incident directly to the responsible team. 

Doing so would take away the pressure to review everything for the service owning team. We can give teams the approval rights to work in other services, so each PR can be reviewed by people having the most context, in minutes rather than days. 

The second bottleneck is context. 

Context is the bread and butter of AI systems, but enterprises are prone to having incomplete or incorrect contexts that inhibit the agent from doing higher quality work. 

I've often read code that's so old that no one knows how it works or why it's there. That context is lost as the person who wrote it left the team. Even the things that are documented can create problems. Some documentation conflicts with each other about how a feature or internal process is supposed to work. These legacy artifacts accumulate through years of iterations and are often not cleaned up, creating confusion for agents. 

Prioritizing documention and other quality work has not been effective. Despite leadership's emphasis on their importance, writing documentation and general bookkeeping don't get people promoted. So they keep getting pushed to make way for shinier and less mentally taxing feature work. 

To be better, we now automatically construct the org's knowledge graph by identifying and encoding each of our team's processes into skills, and use agents to document things done at every step onto our sprint (or work tracking) board. We built the workflow on top of Matt Pocock's engineering workflow and its [ecosystem of skills](https://github.com/mattpocock/skills).

Let's go thorugh 3 examples for implementing small features, large features, and being oncall after adapting this workflow.

Feature work comes to our team with specifications written by PMs. Smaller features are those that take 1-3 months, pre AI, to complete. The engineer usually has a rough idea on how to write them, or that there are existing features with similar structure that we can reference.

We use /grill-me to establish a shared understanding with the agent on what and how to build. /to-spec to turn that understanding into a design doc that goes into details about all the REST endpoints, model entities, and interfaces needed. /to-tasks to breakdown the spec to a tree of tasks with hierarchies baked in. Some can be implemented in parallel, some can be worked on after the predecessor tasks are completed. 

Throughout this process, we use /issue-tracker, perhaps the most important skill, to document things on our sprint board as deliverables and tasks. They can be later queried and understood by agents in investigations. 

Later, /implement spins up multiple agents, each grabbing a task ready to be worked on. They might create pull requests, which are linked back to the task to complete the knowledge graph construction. 

Bigger features are ones that might take anywhere from 3-12 months to complete. We've had to work on a new protocol or add infra for certificate delivery that touches on plenty of areas outside of our team's expertise. 

For these, we use /investigate to explore code bases, gather context, and make decisions, all while documenting each step with /issue-tracker. We chain together /grill-mes and /zoom-outs and /ask-experts with /investigate being the top level coordinator that keeps track of the state. It creates a stateful representation of what we've learned and what remains to be explored on our sprint board so that a future agent can pick up right where another one left off, a process common in the months long investigation and design process. 

Once the investigation's done, /to-spec turns hundreds of items documented into a design doc, and the rest of the flow remains the same as smaller feature work.

For incidents, we use /icm-triage to root cause. It looks at /incident-history, which consists of items created from previous runs of /icm-triage, and if it identifies the problematic feature, the deliverables and their associated tree of tasks made from the /implement of the feature. It also has the ability to search through our code, look at metrics, and query telemetry. 

85% of the time, it correclty identifies the root cause, emits a report containing raw artifacts like code pointers, queries ran, log lines from those queries, so that each decision can be independently verified. The oncall engineer is in charge of making sure this report is accurate, or to iterate with the agent if not. 

Once done, /icm-triage documents all investigation context with /issue-tracker. 

Features developed with /implement are true AI native features. All the decisions made together by the engineer and AI are captured in a place friendly to both humans and agents, making them incredibly easy to service. 

The last bottleneck is coordination. 

[A company is a vector of all its people.](https://x.com/elonmusk/status/1871997501970235656?s=20) The bigger the company, the harder it is to align all the people in the same direction. In practice, this looks like different teams having different AI adoption speed and patterns. Some run advanced workflows that automate 90% of incident triaging. Others copy and paste code into AI chatbots to get answers. 

The cost of not standardizing workflows and context collection is that the agent misses out on compound improvements. Since the AI itself is a centralized brain, connecting previously missing context anywhere improves the accuracy of workflows everywhere. 

It's important to relentlessly centralize the distribution of AI related artifacts. Make a marketplace that bundles each set of skills as plugins so that team members no longer need to copy and paste MD files back and forth in chats. Invest in a one time setup so that everyone has the required plugin installed through a one time setup, so that in the future they get all updates each e they boot up their CLI. 

If the skills are discoverable, the correct ones will get invoked automatically. The user doesn't need to keep track of the constantly changing skills catalog. They'd just be pleasantly surprised at how the agent can now handle an area that they didn't think it could before. And if they notice something's not working, they'd be prompted to make improvements or file issues so that the system improves for everyone else once fixed. 
