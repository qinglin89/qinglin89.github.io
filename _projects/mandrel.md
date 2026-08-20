---
layout: page
title: mandrel
description: An open-source protocol for running AI coding agents on the same codebase for months — bounded working sets over unbounded project time.
img: assets/img/projects/mandrel-cover.jpg
og_image: https://qinglin89.github.io/assets/img/projects/mandrel-cover.jpg
importance: 1
category: work
related_publications: false
---

mandrel is the protocol underneath the way I do AI-assisted development: task-scoped work,
selective project memory, and deterministic control outside the model, deployed into every
repository I work in. It is open source under Apache-2.0, and it is the layer
[orch-hub](/projects/orch-hub/) sits on top of — orch-hub operates a portfolio of repositories,
mandrel is what each of those repositories actually runs.

[Read the full write-up →](/blog/2026/context-isnt-the-bottleneck-drift-is/) ·
[View the repository on GitHub →](https://github.com/qinglin89/mandrel)

#### The problem is drift, not context length

The failure mode of long-horizon AI-assisted development is not model capability. It is
**drift**: decision history accumulating without curation until neither the human nor the
agent can face it. Note the axis — drift is a function of project *time*, not project size. A
hundred thousand lines generated in a week has almost none; twenty thousand lines iterated
over two years is where it becomes lethal. What accumulates is not code, it is the sediment of
choices made and reversed.

Longer context windows do not fix this. The window fills with the transcript of how you got
somewhere rather than the conclusions, and capacity was never the binding constraint —
precision is.

#### One invariant

> **The working set stays bounded over unbounded project time.**

Day 1 and day 300, a fresh session faces the same *shape* of context: a small constant set of
project invariants, one task, and a routed handful of relevant documents. The corpus behind it
grows. The slice loaded into any given session does not.

#### Three mechanisms

**Tasks are sized to one context window.** A task that fits in one session gets completed; a
task that doesn't gets handed off, and every handoff loses information. The protocol's job is
not to make handoffs lossless — that is unachievable — but to make them rare and structured.
The task file carries the full lifecycle: development, fresh-context review, remediation,
handoff, and completion. Review converges through frozen finding groups, delta-only re-review,
and one-shot human escalation.

**Memory admits rather than accumulates.** Writes to the project snapshot happen only at task
completion, and a fact must pass three tests to enter: is it expensive to re-derive, does it
stay true, and does knowing it change what the agent does next. What passes are invariants,
non-obvious couplings, anti-patterns, and *intentional omissions* — the category no amount of
reading recovers, because absence leaves no trace in the code.

**Deterministic control lives outside the model.** A caller re-parses the task file, selects the
next legal role, assembles and injects context and contracts, verifies declared outputs, counts
convergence budgets, and escalates decisions it may not make.

#### What the protocol does not trust

The suite is layered by a single question: can a model decline to comply? Contracts, role
specifications, and skills are semantic — they work by being read and understood, which means
they work most of the time. Session hooks and the scheduler's output verification are
mechanical: they run outside the model's judgment and cannot be reasoned with.

That split is the design. The semantic layers carry everything requiring judgment, because
only a model can supply it. The mechanical layers carry the few properties that must hold
regardless — the tree is clean, the handoff exists, the declared output is really there — and
they are deliberately few, because every mechanical check is a rule you can no longer change
by argument.

#### Two interchangeable executors

Role contracts are anonymous and self-contained: a session never learns that other kinds of
session exist, and the caller owns all sequencing. That is what lets a human run the loop by
hand and a headless scheduler run it unattended, from the same runbook, with byte-equivalent
delivery — you can take the wheel at any session boundary and the scheduler picks up where you
left off. Automation you cannot step into is automation you must trust completely or abandon
completely.

#### In use

Five months across four repositories, 286 completed tasks, the largest a ~122k-line Go service
whose curated snapshot sits at ~34k words. The protocol was introduced to that codebase ten
months after its first commit — a brownfield adoption, which is the situation most readers are
actually in.

**Stack:** Markdown contracts · Python deployment and drift-detection CLI · Bash session hooks
for three agent surfaces · a headless dev/review scheduler · a deterministic verification gate.
