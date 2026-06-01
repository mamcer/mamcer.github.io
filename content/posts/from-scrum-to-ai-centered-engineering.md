---
date: 2026-05-31
layout: post
title: From Scrum to AI-Centered Engineering
subtitle: When Coding Stopped Being the Bottleneck
---

The presentation that started all of this opened with Egyptian hieroglyphics next to the word "Scrum." Not as a joke. As a serious observation: Scrum was invented to manage a specific bottleneck, and that bottleneck is gone.

For a while we ignored this. The team knew something was off but we kept running sprints, estimating in points, tracking velocity. Then the data started getting hard to ignore. Features that had always been "5 points" were shipping in a few hours. Our velocity numbers were climbing but they had stopped meaning anything. A sprint with a lot of Claude-assisted bugfixes could score higher than a sprint where we shipped something genuinely hard and both would read the same on the chart.

The problem wasn't Scrum exactly. The problem was that Scrum optimizes for implementation speed, and implementation speed had stopped being our constraint. Code was no longer the bottleneck. Review was. Coordination between services was. Getting things actually deployed was.

So we rebuilt the methodology from scratch.

This is a story that is still being written. Like any cultural and methodological transformation, it takes time, iteration, experimentation, and learning from failure. What you will see here is only part of our journey, with challenges still to solve and opportunities yet to explore. At the same time, the results already show that this is not a short-lived spike. It is a lasting trend that we are committed to driving forward.

## What we changed

The new framework pulls from a few sources: DORA Metrics for delivery health, the SPACE Framework from GitHub and Microsoft Research for understanding developer productivity without collapsing it to a single number, Mik Kersten's Flow Framework for making bottlenecks visible, and Shape Up from Basecamp for the core planning concept. **None of these are new ideas**. The combination, and the way they interact with AI-assisted development, is what we had to figure out for ourselves and we are continue evolving.

The biggest operational change was replacing estimation with what Shape Up calls appetite. The question isn't "how long will this take?" it's "how much is it worth investing in this?" These feel similar but they produce completely different answers. With estimation, scope is fixed and time adjusts: the feature is what it is, and if it takes longer than expected, the sprint slips. With appetite, time is fixed and scope adjusts: we decide upfront how much investment makes sense given the business value, and if the technical solution is too large, the solution gets smaller. This was a big change for us.

The second change was formalizing something we had been trying to pretend didn't exist: our backend and frontend have fundamentally different delivery cadences. The backend can deploy daily, a merged PR can be in production within minutes. The mobile app is gated by App Store and Play Store review processes that take 1–7 days. Any process that tries to run both tracks on the same rhythm creates friction. We stopped pretending they were the same and made the difference explicit in how we plan.

We also got serious about WIP limits. Little's Law isn't a suggestion: lead time equals WIP divided by throughput. If throughput is constant and WIP doubles, lead time doubles. With AI tooling accelerating implementation, the real failure mode became opening four PRs in parallel, each one taking 30 minutes to write, and having all four sit in the review queue simultaneously. High WIP is the cause of slow delivery, not a symptom of it. The current limits are two items per person in the fast lane (bugs, hotfixes, chores), one active feature per person, and two PRs open for review at any time. When someone hits the ceiling, the right move is to review before opening something new.

The fourth change was treating our CLAUDE.md files as a technical contract rather than decorative documentation. Every repository has one. It documents the things that aren't obvious from reading the code: which service contracts can't change without cross-team coordination, which patterns are intentional, which domains have constraints that aren't visible locally. Updating it is part of the definition of done for any PR that introduces something new. The reason is that without explicit context, AI tools generate plausible-but-potentially-wrong code. Code that works in isolation but can break something downstream.

Finally, we automated measurement from day one. We built a pipeline that ingests git history and Azure DevOps pull requests into MySQL, with Metabase dashboards over DORA and Flow metrics that update daily. Nobody on the team manually tracks anything.

## What the data shows

The most direct number: Q1 2025, the team merged 310 pull requests. Q1 2026, it was 694. That's 2.24 times the output with the same seven engineers. No headcount increase.

March was the highest output month in the project's history. The growth is consistent, not a spike.

Deployment frequency went from 0.90 deploys per day in November to 1.70 in March. The DORA elite benchmark for this metric is "multiple times per day", we are approaching it from below 1 less than six months ago.

On lead time, the average PR went from 48.9 hours in Q4 2025 to 4.6 hours in Q1 2026. Some of that is the WIP limits keeping the review queue from stacking up. Some of it is that appetite-based scoping prevents oversized items from entering the board in the first place.

We also track AI adoption directly, because adoption varies and the variance matters. The main signal we watch is micro-commit percentage, commits under 200 lines, which reflects whether someone is working in short AI-assisted iterations or still doing large batch sessions.

## What is still broken

Our Change Failure Rate metric, the percentage of deployments that cause an incident, uses hotfix and revert patterns in PR titles as a proxy for failures. The detection is currently too broad. It catches naming patterns that aren't actual failures, producing numbers that clearly don't reflect reality. We are refining the logic.

Review wait time, how long a PR sits before the first review comment, is still measured manually. We know review is now the primary bottleneck. We can't quantify it in real time yet.

Real MTTR (mean time to restore after an incident) requires logging the moment of detection, which today lives in IM and War Room conversations, not in a queryable system.

## A note on what we learned about AI tooling specifically

The throughput number is real, but there is a version of this story that goes badly: AI generates speed without judgment, review rates drop because there is too much to review, and change failure rate climbs. We watched for this.

The team of 7 doubled its output. Not by working more hours. By managing the constraint that actually exists now.

This journey is still far from over.