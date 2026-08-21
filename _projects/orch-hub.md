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
The repository is private; this page describes what it is and why it exists.

The protocol those loops actually run is [mandrel](/projects/mandrel/), which is open source.
The division is deliberate: mandrel is what a single repository runs and is useful on its own,
with or without a control plane; orch-hub is the layer above it, where a portfolio of
repositories becomes one operating surface.

#### The product in use

The dashboard keeps the portfolio legible at a glance: repository readiness, live-run state,
and active-task counts share one operating surface. A waiting run is deliberately more visible
than an idle one, because it is the place where human attention can unblock the system.

<figure style="margin:1.5rem 0;">
  <img src="{{ '/assets/img/projects/orch-hub-repos-ui.jpg' | relative_url }}"
       loading="lazy" alt="orch-hub repository dashboard showing readiness, live-run state, and active-task counts"
       style="display:block;width:100%;border-radius:10px;box-shadow:0 8px 30px rgba(0,0,0,.15);">
  <figcaption style="margin-top:.6rem;color:var(--global-text-color-light);font-size:.9rem;">
    Portfolio view — ready repositories, bounded task inventories, and a run waiting for input.
  </figcaption>
</figure>

When a run reaches a decision boundary, the task view exposes the question in context. The
operator can discuss a change or give a binding answer from the same phone-first surface; the
agent continues afterward without turning the whole workflow back into a terminal session.

<figure style="margin:1.5rem 0;">
  <img src="{{ '/assets/img/projects/orch-hub-approval-ui.jpg' | relative_url }}"
       loading="lazy" alt="orch-hub task view showing a waiting plan gate with Feedback and Confirm actions"
       style="display:block;width:100%;border-radius:10px;box-shadow:0 8px 30px rgba(0,0,0,.15);">
  <figcaption style="margin-top:.6rem;color:var(--global-text-color-light);font-size:.9rem;">
    Human-in-the-loop at the right altitude — review the plan, send feedback, or confirm.
  </figcaption>
</figure>

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
