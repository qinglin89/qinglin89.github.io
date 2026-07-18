---
layout: page
title: orch-hub
description: A local control plane for AI coding agents — one operator, a portfolio of repos, and year-long projects kept inside window-sized tasks.
importance: 3
category: work
related_publications: false
---

orch-hub is a control plane I design and build for my own AI-assisted development. It runs
resident on my Mac, is operated from a phone over a private network, and coordinates
per-repository orchestrator programs that drive autonomous dev/review loops over task files.
The repository is private; this page describes what it is and why it exists.

#### The unit is a product lifecycle, not a chat session

A production-grade project is rarely one or two agent sessions — from idea to launch it is
months of work across hundreds of sessions. Most AI coding surfaces manage the single session
well and leave the *time* axis to the human. orch-hub manages the layer above: which repos are
ready, what each backlog holds, which runs are waiting on a human answer, what happened while
you were away — the operating state of a whole portfolio, not a list of conversations.

#### Bounded working set over unbounded project time

The hub fronts a protocol stack — a task lifecycle plus a curated, machine-parsable project
memory — built around one invariant: **the working set stays bounded while the project grows
without bound**. Every task is sized to one context window; completed work passes an admission
gate that distills durable knowledge into a timeless snapshot and archives the rest. The agent
in month twelve faces the same bounded shape as the agent on day one — and so does the
operator: the interface converges instead of accreting.

#### Human in the loop, at the right altitude

Agents run autonomously inside their task; the human is pulled in only at escalation points —
a question, a plan gate, a review verdict, a new-task intake. Those arrive as push
notifications and one-tap answers on a phone, so steering a fleet of long-running agents fits
into the gaps of a day rather than demanding a terminal.

**Stack:** Python · FastAPI · file-contract IPC with detached orchestrator subprocesses ·
mobile-first dependency-free SPA · launchd resident service · push notifications ·
private-network-only, fail-closed bearer auth.
