---
layout: page
title: orch-hub
description: A local control plane for AI coding agents — one operator, a portfolio of repos, and year-long projects carried through bounded development sessions.
img: assets/img/projects/orch-hub-cover.jpg
importance: 4
category: work
related_publications: false
---

orch-hub is a control plane I design and build for my own AI-assisted development. It runs
resident on my Mac, is operated from a phone over a private network, and coordinates
per-repository orchestrator programs that drive autonomous dev/review loops over task files.
The source repository remains private, while a public Apple Silicon binary preview now makes
the operating experience available without a source checkout.

The protocol those loops actually run is [mandrel](/projects/mandrel/), which is open source.
The division is deliberate: mandrel is what a single repository runs and is useful on its own,
with or without a control plane; orch-hub is the layer above it, where a portfolio of
repositories becomes one operating surface.

[View the binary preview →](https://github.com/qinglin89/orch-hub-releases)<br>
[Watch the 19-second walkthrough →](https://github.com/qinglin89/orch-hub-releases#see-the-workflow)<br>
[View mandrel on GitHub →](https://github.com/qinglin89/mandrel)

#### The product in use

The dashboard keeps the portfolio legible at a glance: repository readiness, active-task
counts, recent outcomes, and subscription capacity share one operating surface. The overview
makes the state of the whole portfolio visible before the operator drills into any one
repository or run.

<figure style="margin:1.5rem 0;">
  <img src="{{ '/assets/img/projects/orch-hub-overview-ui.png' | relative_url }}"
       loading="lazy" alt="orch-hub isolated demo operations overview showing repository readiness, active tasks, recent outcomes, and subscription capacity"
       style="display:block;width:100%;max-width:900px;margin:0 auto;border-radius:10px;box-shadow:0 8px 30px rgba(0,0,0,.15);">
  <figcaption style="margin-top:.6rem;color:var(--global-text-color-light);font-size:.9rem;">
    Isolated demo: portfolio readiness, active tasks, recent outcomes, and subscription capacity.
  </figcaption>
</figure>

Run history stays organized around durable tasks rather than isolated conversations. Each task
groups the development and review sessions that advanced it while retaining wall-clock time,
peak context use, launch configuration, and a path to the resulting evaluation.

<figure style="margin:1.5rem 0;">
  <img src="{{ '/assets/img/projects/orch-hub-run-history-ui.png' | relative_url }}"
       loading="lazy" alt="orch-hub isolated demo run history grouping development and review sessions by durable task with context use and evaluation links"
       style="display:block;width:100%;max-width:900px;margin:0 auto;border-radius:10px;box-shadow:0 8px 30px rgba(0,0,0,.15);">
  <figcaption style="margin-top:.6rem;color:var(--global-text-color-light);font-size:.9rem;">
    Isolated demo: task-level grouping, bounded sessions, context use, and evaluation links.
  </figcaption>
</figure>

The public repository distributes preview binaries, installer metadata, screenshots, and the
walkthrough under explicit preview terms; it is not a source-code mirror. That boundary lets
people evaluate the control-plane experience while the protocol and its documentation remain
fully open in mandrel.

#### The unit is a product lifecycle, not a chat session

A production-grade project is rarely one or two agent sessions — from idea to launch it is
months of work across hundreds of sessions. Most AI coding surfaces manage the single session
well and leave the _time_ axis to the human. orch-hub manages the layer above: which repos are
ready, what each backlog holds, which runs are waiting on a human answer, what happened while
you were away — the operating state of a whole portfolio, not a list of conversations.

#### Bounded working set over unbounded project time

The hub fronts a protocol stack — a task lifecycle plus a curated, machine-parsable project
memory — built around one invariant: **the working set stays bounded while the project grows
without bound**. Development sessions are bounded to one effective context window; tasks that
need more than one carry an explicit plan and structured handoffs. At completion, admitted
durable findings update the timeless snapshot and the complete task record moves to the local
archive. The agent in month twelve faces the same bounded shape as the agent on day one — and
so does the operator: the interface converges instead of accreting.

#### Human in the loop, at the right altitude

Agents run autonomously inside a task; human attention is requested at explicit decision
boundaries — a question, an optional plan gate, an intake confirmation, or a convergence or
dispute escalation. Bark can notify the operator when a run needs input or ends, an intake
draft is ready, or a semantic evaluation finishes; answers are given from the phone-first
dashboard.

**Stack:** Python · FastAPI · file-contract IPC with detached orchestrator subprocesses ·
mobile-first dependency-free SPA · launchd resident service · push notifications ·
private-network-only, fail-closed bearer auth.
