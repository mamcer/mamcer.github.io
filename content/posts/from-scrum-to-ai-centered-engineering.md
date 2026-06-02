---
date: 2026-05-31
layout: post
title: AI Didn't Replace Our Bottlenecks. It Moved Them.
subtitle: From Scrum to AI-Centered Engineering
---

In just a few months, a 7-person team doubled its output. Not by working longer hours, but by rethinking how we build software products with AI.

This is a story that is still being written. One stop in a journey. Like any cultural and methodological transformation, it takes time, iteration, experimentation, and learning from failure. What you will see here is only part of our journey, with challenges still to solve and opportunities yet to explore. At the same time, the results already show that this is not a short-lived spike. It is a lasting trend that we are committed to driving forward.

## From AI helpers to AI-driven development

We evolved from using AI for isolated tasks such as refactoring, unit testing, and chat-assisted knowledge support (think Stack Overflow on steroids) to fully embracing AI-driven agentic development. AI is no longer a sidekick, it has become a default part of how we build software. Supported by a team of professionals fully committed to this transformation.

## When Scrum stopped being enough

As we progressed on this journey, our existing methodology—mature, widely adopted, and continuously refined—started showing signs of friction. Scrum wasn't the problem. Coding speed was no longer the bottleneck. The bottleneck had moved: 
- Code reviews
- Cross-service coordination
- Deploying changes into production 

For a while we tried to ignore it, then we rebuilt from scratch.

## The new methodology

We combined several proven frameworks:

- DORA Metrics to track delivery health automatically
- SPACE Framework to understand productivity beyond a single number
- Flow Framework to make bottlenecks visible
- Shape Up for planning in an AI-assisted environment

None of these ideas are new. What is new is the context: a team where AI-assisted development is the default, not an experiment.

## Key changes include:

Estimation replaced with appetite. Not "how long will this take" but "how much is it worth investing." With estimation, scope is fixed and time adjusts. With appetite, time is fixed and scope adjusts. One planning conversation upfront replaces a whole category of sprint carry-over. A big change for us.

Two delivery tracks made explicit. Our backend can deploy daily. Our mobile app is gated by App Store review cycles of 1–7 days. Any process that pretends these are the same cadence creates friction. We stopped pretending.

WIP limits treated as system rules: lead time = WIP / throughput. High WIP is the cause of slow delivery, not a symptom. When someone hits the limit, the right action is to review before opening something new.

Automated DORA metrics from day one: Git commits and PRs feed into MySQL nightly. Metabase handles the rest. No manual tracking.

CLAUDE.md files as living technical contracts. Every repo has one. They document cross-service constraints, patterns, and invisible knowledge, essential in an AI-first workflow.

## The results so far:

Q1 2025: 310 PRs merged. Q1 2026: 694. Same team.

Deployment frequency went from 0.90/day to 1.70/day. Average lead time went from 48.9 hours to 4.6 hours. The DORA elite benchmark for this metric is "multiple times per day", we are approaching it from below 1 less than six months ago.

March 2026 was the highest output month in the project's history.

The same team can now deliver roughly twice the amount of product work it could a year ago.

Not by working more hours.

By removing the constraints that emerged once coding stopped being the bottleneck.

## What's still in progress:

Some things are still broken. Our change failure rate proxy over-detects. Review wait time is still manual. Real MTTR requires logging the moment of detection, today it lives in IM and war rooms, not in a queryable system.

## A note on AI tooling

Throughput is real, but speed without judgment is dangerous. AI can generate code faster than humans, but without proper review, change failure rates could rise. We have been careful to manage the new constraints, not just chase raw output.

The biggest lesson so far for us is that AI didn't eliminate constraints. It changed where they live.

This journey is far from over.