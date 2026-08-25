---
layout: post
title: "Context isn't the bottleneck. Drift is."
date: 2026-08-20 10:00:00
description: Introducing Mandrel, an open-source protocol for long-horizon AI coding, and what 286 tasks across four repositories taught me about drift.
og_image: https://qinglin89.github.io/assets/img/projects/mandrel-cover.jpg
tags: ai-coding agents protocol open-source
categories: engineering
toc:
  sidebar: left
  collapse: expanded
---

## The month-four problem

Four months into a project, a backend service somewhere north of a hundred
thousand lines, I opened a fresh agent session to fix a retry loop that had
spun 22,000 times in fourteen seconds.

The fix was twenty lines. The question that mattered was the one before it:
what does this session need to know?

Not "what's in the file" — the agent can read the file. It needed two things
that were not in the code.

It needed to know that the resync design it was about to delete was deliberate
and correct, and that a previous task had already ruled on it.

It also needed to know that the idempotency angle — which looks like the obvious
fix — is a dead end, because the server's cache cannot separate "executed" from
"confirmed". Nobody decided that. Somebody spent a cross-service investigation
finding it out.

Neither is going to come back by reading harder, for different reasons, and
preserving them buys different things.

The resync **decision** left no trace. Nobody writes down the design they
didn't build, so no amount of reasoning over the code recovers it, at any
model capability. Keeping it is what stops a session from being wrong.

The idempotency **conclusion** did leave a trace — it is implied by the code,
and a session willing to read the server cache would get there. But that is a
cross-service investigation no twenty-line retry fix is going to fund. Keeping
it changes where the next session starts: it begins from that conclusion rather
than from the cache, and the depth behind the conclusion keeps growing while the
context holding it does not. Over months, that is what makes a project's
reasoning cumulative instead of repetitive.

There are three common answers to how you keep that around, and I have watched
all three fail.

**Keep the conversation going.** Long threads, resumed sessions, a context
window measured in millions. The window fills with the transcript of how you got
somewhere while the conclusions become harder to find. In my runs, capacity has
not been the binding constraint; precision has. And a session that must
re-derive before it can extend never gets past the first layer, however large
the window.

**Put it in `CLAUDE.md` / `AGENTS.md` / a memory bank.** Better — now it
survives the session. But without an admission and removal policy, every one I
used or watched closely — including my own first three attempts — converged to
the same end state: a file that only grows, that nobody deletes from because
deleting requires knowing what's still true, and that eventually costs more to
read than it saves. A log is not a memory. A log with an append-only policy is
a log with extra steps.

**Front-load a spec.** Spec-driven frameworks — spec-kit, BMAD, OpenSpec — are
strongest at turning a vague idea into an executable plan and managing
deliberate change. They helped me answer "how do I begin?" What I still lacked
was an answer to "it's month four, how do I continue?" The interesting state
then is everything learned since the initial plan, including decisions that no
longer appear in the current spec.

The failure mode I kept hitting across all three has a name, and it isn't
context length. It's **drift**: what the project has already worked out
accumulating without curation until neither the human nor the agent can face it.
You can feel it happening. Sessions get longer. The agent re-derives things it
derived last week. You start prefacing every prompt with a paragraph of
"remember that we decided…". Eventually you stop trusting the output and read
every diff yourself, which is where the productivity you bought disappears.

**Drift follows project time more than project size.** A large codebase
generated in a week can carry less drift than a much smaller one iterated over
two years. What accumulates is not code but the sediment of what was settled,
reversed, and worked out along the way.

Preserving those two things is not a filing problem. Both have exactly one
origin — a task that finished and was reviewed — and exactly one consumer: the
next task, which starts from the memory as it currently stands. Memory without
the task lifecycle has no admission point and no provenance; the task lifecycle
without memory starts from zero every time. Neither half works alone, and that
is why what follows is a workflow rather than a document format.

For the past five months I have been building and running the thing I actually
wanted. Today I am open-sourcing it as **Mandrel**: an Apache-2.0 protocol and
small toolchain for running coding agents on the same repository for months. It
uses versioned Markdown contracts, a Python/Bash deployer, and an optional
scheduler, with support for Claude Code, Cursor, and Codex. I have used it
across four repositories and 286 completed tasks, including a ~122k-line Go
service.

[Mandrel on GitHub](https://github.com/qinglin89/mandrel) ·
[Quickstart](https://github.com/qinglin89/mandrel#quickstart)

This is what I learned. If you only want the smallest useful piece, jump to
[Which part is load-bearing](#which-part-is-load-bearing).

## The invariant

The invariant is one line:

> **The working set stays bounded over unbounded project time.**

Day 1 and day 300, a fresh session faces the same _shape_ of context: a small
constant set of project invariants, one task, and a routed handful of relevant
documents. The corpus behind it grows. The slice is bounded by the task rather
than being allowed to grow merely because the project is older.

I implemented it with three mechanisms: task-scoped growth, selective memory,
and deterministic control outside the model.

Bounded, though, is only half of what matters. The other half is what the set
contains: the decisions and conclusions from the opening story. The first keeps
a session from being wrong; the second lets it begin reasoning at a depth it
could not afford to reach from zero.

I implemented it with three mechanisms: task-scoped growth, selective memory,
and deterministic control outside the model.

A project evolves task by task. Each task produces both a code change and a memory
update. Within a task, sessions hand off through the session log.
Across tasks, the memory system carries what survived. And an orchestrator decides
what runs next and mechanically checks that it happened.

One principle generates all three: anything that can be decided without model
reasoning should be guaranteed by deterministic code outside it. The model supplies
judgment. Everything mechanically checkable — did the entry get written, is the
status transition legal, is the tree clean — is checked where a persuasive
explanation cannot waive it.

Two directories carry the project-owned state. **`.ai/`** is curated memory: a
timeless, version-controlled snapshot of how the system currently is. Ordinary
work sessions never write it; durable findings enter through task-completion
closeout, while explicit housekeeping may restructure it. **`.ai-tasks/`** is
local, gitignored in-flight work: one file per task, each accumulating a session
log that carries state across sessions. Its archive is therefore a local audit
trail unless you back it up separately. A scheduler I'll call the
**orchestrator** runs the loop over one task headlessly between human
escalations, or a human runs the same loop by hand.

```text
   .ai/  ───────────────┐          ┌──────────────────────────┐
   curated memory       ├─────────►│                          │
   eager set: project   │          │       ONE SESSION        │──► commits
   invariants + routing │          │   fresh context, ~200k   │
   routed: the docs     │          │                          │──► session-log
   this task named      │          │  works under exactly one │    entry/entries
                        │          │  role contract — dev or  │
   .ai-tasks/<id>.md ───┘          │  review — that names no  │
   goal · scope ·                  │  other role              │
   acceptance ·                    └──────────────────────────┘
   session log                       ▲                    │
                                     │ assembles          │ verifies
                                     │                    ▼
        ┌────────────────────────────┴───────────────────────────────────┐
        │  ORCHESTRATOR — outside the model, no durable lifecycle state  │
        │  re-parses the task file every turn · builds the role prompt · │
        │  verifies declared outputs · counts convergence budgets ·      │
        │  escalates to a human on anything it may not decide itself     │
        └────────────────────────────────────────────────────────────────┘

   on completion: absorption distils the session log into .ai/,
   the task file moves to archive, and the loop ends.
```

Every dev or review session's durable declarations go in the task file. Durable
findings enter `.ai/` at closeout, and only if they pass the admission tests
below. The eager substrate arrives through tool hooks or imports; task
frontmatter routes the additional documents the session reads.

## Mechanism 1: the task is the unit the project grows in

Every unit of work is a file. Frontmatter carries status, blockers, and a
session estimate; the body carries Goal, Scope, Acceptance, and an appended
session log.

The file is more than work tracking; I use it for four jobs at once:

- **It bounds an increment.** Goal, Scope, and Acceptance are a contract for one
  step of growth: what changes, what deliberately doesn't, and how you know it
  landed. The project advances as a sequence of scoped, accepted steps rather
  than as a stream of commits.
- **It carries state across sessions**, with a sizing constraint designed to
  keep those handoffs rare.
- **It is where ongoing memory comes from.** After brownfield initialization,
  absorption reads the accumulated session log at completion and distils it.
- **It is the local audit record.** Status transitions, which session claimed
  what, review verdicts, unresolved disputes, the convergence group a finding
  belongs to. How a change was arrived at survives in a form you can read back
  on that checkout, not just what the diff ended up being.

The estimate is denominated in a specific unit:

> one development session ≈ one effective context window (~200k tokens)

A task estimated `0/3` claims its development work needs roughly three windows;
review sessions do not consume the estimate. If it turns out to need five, the
estimate is raised and the plan re-sliced mid-flight. The estimate is not a
schedule. It's a **sizing constraint**, and it exists because of an asymmetry I
underestimated at the start:

**Development work that fits in one session avoids a handoff. Work that does
not fit gets handed off — and in practice every handoff sheds information.**

The outgoing session knows a hundred things it never writes down, because it
doesn't know which ninety-nine are irrelevant. So the protocol's job is not to
make handoffs lossless — that's unachievable — but to make them **rare and
structured**: rare by sizing work to fit, structured by specifying exactly what
a handoff must carry.

Per development entry, three required fields plus the status transition:
**Done** (what happened — committed changes, decisions made _including rejected
alternatives and why_, facts learned worth recording), **Next** (remaining
work), and **Open** (unresolved questions). **Plan-slice** is optional. Review
entries instead carry **Verdict**, **Group**, and **Findings**.

The one that earns its keep is _rejected alternatives_. Here is the actual
Scope from that retry-loop task — lightly anonymized, otherwise as written:

```markdown
## Scope

- `submit` must not call `resync` directly on error. Return to the main loop
  instead: if the decision did land, the engine emits the next step and the
  client continues normally; if it did not, the read times out and the existing
  stall branch calls `resync`. That branch already covers both cases, so the
  direct call is redundant — and harmful, because it bypasses any step already
  sitting in the local FIFO.

Out of scope, deliberately:

- **The resync design is correct and stays.** Reading the newest emitted step
  and catching up with an empty decision is the right response to "the engine is
  one step ahead" — it self-heals whenever the emitted step propagates. Do not
  replace it.
- Reusing the decision id across retries does nothing on its own: the
  idempotency cache cannot help until the server separates "decision executed"
  from "apply confirmed". Server-side concern.
- The trigger itself is NOT addressed here and is NOT understood — an apply
  barrier timed out while, in the same instant, the step the engine had just
  emitted stopped propagating. Two independent downstream paths stalled together
  for ≥19s. Service logs for that window are gone. Fixing this task does not fix
  that; it only ensures the client fails loudly and quickly instead of spinning.
```

Three of those four bullets are about what **not** to do. That's the expensive
knowledge — the part a fresh session would spend an hour rediscovering, or
worse, would not rediscover and would confidently undo. The fix was twenty
lines. The Scope is the artifact.

The rule is: **the task file is ground truth, and no cross-session decision may
derive from conversation memory.** Not from what I remember telling the agent.
Not from the scrollback. If it isn't in the file, it does not exist. That is
what makes sessions genuinely interchangeable.

And when a task completes it does not evaporate. Absorption takes whatever
passes the tests into the snapshot; the file itself moves to an archive, sorted
chronologically by name and greppable. **That two-tier split is what lets the
working set be bounded without history being thrown away.** The snapshot stays
timeless because the archive is allowed to be temporal — everything that was
true only for a while, every rejected route not durable enough to admit, every
review verdict, is still there, just demoted out of what a session loads by
default. Bounded means _demoted_, not _discarded_, and those are very different
promises to make about a project you intend to work on for a year.

That archive turns out to earn its keep in a second way I did not build it for.
Three months on you are a stranger to your own decisions, and the thing you
actually want then is not a chat log — it is the scoped increment, what it
accepted, what the review said, what stayed unresolved. It is also what makes
delegation survivable: you can hand work to an agent and check afterwards at the
level of decisions rather than reading every diff, which is the only version of
"trust it" that is not just hoping.

## Mechanism 2: memory admits, not accumulates

`.ai/` is a small directory describing the project as it currently is.
Timeless: no changelog, no "we used to do X", no dated decisions. Architecture,
design principles and their tradeoffs, module topology, conventions, a routing
index, a feature-to-module map.

The format matters less than the **write policy**, which has two halves.

**First half: knowledge admission happens at exactly one moment.** Not during
work. A session that notices `.ai/` is wrong does not fix it — it records the
discrepancy in its session log and moves on. Absorption happens at task
completion, as a distinct step, reading the task's whole session log at once.
Explicit housekeeping may later split or reroute snapshot documents, but it
does not turn a mid-task belief into durable knowledge.

I initially thought this was pointless ceremony. It is now one of the rules
I'd defend hardest, because **a mid-task session is the worst possible author of
durable knowledge.** Its model of the work is local, partial, and still
changing; what it believes at hour two may be wrong by hour five. Deferring the
write to completion means what gets absorbed is a _finished_ understanding,
written by a reader of the whole arc rather than a participant in one leg of
it.

**Second half: a fact must pass three tests to get in.**

| Test                | Passes when                                                                              |
| ------------------- | ---------------------------------------------------------------------------------------- |
| **Derivation cost** | Re-deriving it requires multi-file traversal, cross-module reasoning, or git archaeology |
| **Stability**       | It stays true across iterations without re-verification                                  |
| **Leverage**        | Knowing it changes the agent's next action                                               |

A clear failure on any test keeps the fact out. If none clearly fails but one is
borderline, compress it to a single keyword-dense line and keep it. Don't
deliberate; deliberation costs more than the line.

The exclusions show what those tests are doing:

- Function signatures fail **derivation cost** — grep is cheaper than a doc.
- "We're currently refactoring auth" fails **stability** — true for two weeks.
- "This project uses PostgreSQL" fails **leverage** — the agent discovers it the
  moment it matters, and knowing it early changes nothing.

What passes: invariants, topology, non-obvious couplings, anti-patterns,
vocabulary, **intentional omissions**, runtime dispatch paths that static
reading won't reveal.

_Intentional omissions_ need special treatment because they are nearly
impossible to recover from code: absence leaves no trace. An
agent cannot distinguish "there is no cache here because it's deliberate" from
"there is no cache here because nobody got to it," and it will helpfully add
the cache. A snapshot recording the _why_ of an absence buys something no
amount of reading can reconstruct.

Here is a real entry from Mandrel's own design document — this example switches
projects from the service in the opening story:

```markdown
### Static Claude imports versus dynamic memory entrypoints

- Chose target-aware deploy rendering for Claude and dynamic hooks for
  Cursor/Codex.
- Consequence: status separately detects stale/ambiguous entrypoints because
  normalized content hashes intentionally treat legal file forms as equivalent.
```

Two sentences. Re-deriving that consequence means reading the deploy renderer,
the status checker, and two hook scripts, and noticing why a hash comparison
cannot see the thing it needs to see. That is what a passing fact looks like.

### Why this resists degradation — and lets depth accumulate

My concern was that repeated distillation would produce a summary of a summary:
a photocopy of a photocopy. The design resists that degradation structurally:
**each round re-grounds against the code, not against the previous round's
conclusions.** A session reads `.ai/`, then works in the actual repository —
running tests, reading diffs, watching things fail. When a stale claim intersects
the code it touches, the contradiction can be recorded for correction at
closeout. The snapshot is a lens on a source of truth that outranks it, not a
replacement for it. It is version-controlled for the same reason: when it is
wrong, that is a reviewable diff.

The same structure lets reasoning depth accumulate. A session starts from code
plus previously reasoned conclusions, so it only pays for the top inferential
layer. The loader entry above took real work to establish: a file can hash
correctly and still not be the thing that loads. Months later, when Mandrel's
workflow skills moved from user-level to per-repository deployment, a fresh
session asked the next question: if personal-level skills take precedence, does
the same category apply? It does. A stale personal copy can win while the
manifest, lockfile, and content hashes all report agreement. That became the
`shadowed skill` drift state.

Getting from "a hash matched" to "hashes cannot tell which of two legal
locations is in force" was the layer already paid for. The second finding cost
one question instead of a rediscovery. **Recording a derived conclusion does
not retire its source**; entries leave when they stop being true, not merely
because another conclusion was built on them. The protocol has no separate
mechanism that computes compounding, and no automatic sweep that prunes old
entries. Compounding is what the admission rule makes possible.

## Inside the task lifecycle: review is a bounded state machine

Development is only one part of a task's lifecycle. Review has to check the
landed work, and the review loop has to terminate. My first version — "have
another agent review it" — produced six consecutive changes-requested rounds
against six remediation sessions. It converged, but not because the loop was
designed to.

Mandrel bounds it with three rules and a budget:

1. **Freeze convergence groups at first review.** A re-review checks only whether
   the original findings are resolved and whether their fixes introduced
   correctness regressions. It may not discover a new backlog in code the first
   review already passed, and a fix session never opens a new group.
2. **Let only correctness block completion.** Design and test findings are fixed
   when cheap or become new tasks; style never blocks. Review stops broken work
   from shipping without holding it hostage until it is ideal.
3. **Escalate a genuine dispute once.** If the reviewer still holds a disputed
   finding after examining it on the merits, it records `Dispute-unresolved:`
   and sends the question to the human instead of requesting the same change
   again.

A group gets two changes-requested re-reviews before a binding human ruling;
the dispute path escalates immediately. Findings remain reviewer judgment. The
state and budgets that make the loop terminate are deterministic policy.

## Mechanism 3: deterministic control lives outside the model

The task and memory mechanisms define state and policy, but something still has
to execute the loop: choose the next legal role, deliver its contract and entry
instructions, verify its declared outputs, count convergence budgets, and stop
for a human when policy has no mechanical answer.

I built a scheduler that runs this loop headlessly using the configured agent
and model for each role. The model supplies judgment; the scheduler owns the
dispatch and checks that do not require it.

### Caller anonymity keeps execution interchangeable

My first version wrote the protocol _for the scheduler_: role documents named
what another session would do, prompts restated rules, and flow state lived in
the caller. It stopped working when I ran a session by hand.

So a session now faces only its own self-contained contract: inputs,
responsibility, and declared outputs. **It never needs to know that other kinds
of session exist.** Fields also have role-local meanings. A remediation session
may report `fix-set: open` because its own fix set is incomplete; only the
caller-side runbook says what that declaration causes next. The session reports
state rather than optimizing for a scheduling outcome.

**The scheduler is a dumb function of the task file.** It persists operational
session provenance for resume routing and holds transient per-run counters and
rulings, but no separate durable lifecycle state. Every iteration re-parses the
file and re-derives what to do next. At ordinary boundaries with no ruling
pending, the loop can stop, resume a week later, or interleave with sessions I
run by hand. The manual slash command delivers the same contract text
byte-for-byte, so I can take the wheel at one of those boundaries and the
machine picks up from the file.

Four litmus tests define the boundary: role documents do not name other roles
or predict the next dispatch, fields are documented by local meaning, and
caller prompts instantiate rather than restate contracts. A lint script catches
the greppable subset; interpretive cases remain review work. Principles that
are not checked decay.

### The enforcement ladder

The protocol is enforced through five layers. They differ in whether the rule is
interpreted by the model or checked outside it.

| Layer                                                             | Where it runs                                            | What it catches                                                             |
| ----------------------------------------------------------------- | -------------------------------------------------------- | --------------------------------------------------------------------------- |
| **Protocol docs** — conduct, schemas, project invariants          | ambient in the model's context                           | a decision made without the invariants in view                              |
| **Role contract** — dev, review, or plan, delivered at invocation | activation prompt; the role-specific imperative contract | doing a different job than the one asked for                                |
| **Skills** — packaged procedures (intake, closeout, housekeeping) | the model invokes them                                   | a multi-step procedure done from memory and half-right                      |
| **Hooks** — session start and stop, in the agent's own runtime    | **outside the model**                                    | a dirty tree after a declared handoff or completion; context-budget wrap-up |
| **Orchestrator** — prompt assembly, output verification, budgets  | **outside the model**                                    | missing or illegal task-file outputs; a loop that will not converge         |

The top three are semantic: they work by being read and understood. The bottom
two execute checks outside the model. Their recovery actions may still require
model judgment, but the model cannot waive a failed check with a persuasive
explanation. In orchestrated mode every formal development and review turn gets
the full post-check set; plan gates, closeout, and read-only discussion have
their own narrower checks. Interactive stop hooks enforce a subset at declared
handoff and completion boundaries.

That split is the design. Semantic layers carry what requires judgment;
mechanical layers carry the small set of observable properties that must hold
regardless — the tree is clean, the entry exists, the declared status is legal.
This is as close as I can get to making an agent follow a protocol without
pretending its judgment can be made mechanical.

## Did the memory bound actually hold?

I could reconstruct the `.ai/` component retroactively: for each of 150
completed tasks in the service from the opening story, how many bytes were in
the task's declared eager-plus-routed `.ai/` slice?

This does not prove that every declared byte reached the model, and it is not
the full session footprint. It excludes the fixed loader, protocol schemas,
role contract, task file, and growing conversation. The figures also exclude
`.ai-tasks/index.md`, which is part of the real eager baseline but is required
only to stay small, with no numeric ceiling. What this measurement can test is
whether the six eager `.ai/` snapshot documents grow with project age, and how
much task-routed `.ai/` material is declared beside them.

Here, "bounded" does not mean constant. The measured memory assembly has two
parts. The **eager set** is what every session pays regardless of task: the
routing index, the map, and the current entrypoint of four core documents. The
**routed** part is what that particular task named in its frontmatter.

The protocol gives those six `.ai/` documents a declared ceiling: a single
document may not exceed ~3000 tokens and an index ~1500, so together they cap
out around **16k tokens**. When one outgrows its limit it is split, and the
overflow moves to the routed tier where only tasks that need it pay. This is a
policy limit, not a continuously enforced runtime gate. The question is whether
the correction happened in practice.

Here is the eager set, sampled across four months:

```text
task    date         six-doc .ai/ eager subset (bytes)
   1    2026-04-10       14,719
  25    2026-05-13       16,288      flat for six weeks
  31    2026-05-19       38,483      climbing
  43    2026-06-06       27,084   ←  a document outgrew its limit and was split
  61    2026-06-12       28,132
  85    2026-07-04       34,972      climbing
 109    2026-07-27       39,689      peak
 121    2026-07-29       32,561   ←  split again
 145    2026-08-14       30,828      flat since
```

It's a sawtooth, and it stays under the declared ceiling. It oscillates between
roughly 4k and 10k tokens — against a cap of 16k — and the correction fired
twice.
Today it sits at about half its ceiling, four months and 150
tasks in. The number of eagerly-loaded `.ai/` documents stayed constant at six
the whole time; only their contents moved.

The routed term behaves differently, and should: it's a function of the task,
not directly of the project's age. A task touching thirteen subsystems names
thirteen documents, on day 1 or day 300. Within this memory-only measurement,
eager plus routed averaged ~5k tokens across the first fifteen tasks and ~15k
across the last fifteen; the heaviest task assembled ~43k, about a fifth of the
200k working budget. Growth, then, but in the term that tracks task width.

Then I ran the same measurement on a second project, 87 tasks, and it is the
more useful warning. Its eager set corrected once, early, then climbed
monotonically to ~14k tokens — **89% of the declared ceiling, with no second
split.** That is not yet a breach, but it leaves little headroom and exposes the
operational weakness: there is no continuous whole-snapshot gate. Closeout can
flag touched documents for manual housekeeping; it does not keep rescanning the
entire snapshot.

So: one project shows the correction firing twice, while another shows how
close the policy can get to its boundary without a global alarm. The invariant
is only as good as its enforcement. These two reconstructions cover 237 of the
286 completed tasks; the other two repositories contribute to the overall
operating total, not to these plots.

## Which part is load-bearing

If you take one thing, take the **admission tests**. Derivation cost,
stability, leverage. They're twenty minutes of work, they need none of the rest
of this, and they convert a memory file from a log into something with an
editorial policy. Apply them to whatever file your agent already reads and
remove anything that clearly fails even one test; compress and keep borderline
cases.

If you take two, add **fresh-context review**. A separate session with no
memory of writing the code is well positioned to catch defects the author
misses. In my runs, even cheaper reviewers did so; fresh context contributes
something capability alone does not.

**Task sizing and role anonymity are the ones you start paying for after the
first two are running.** They matter — sizing is what keeps handoffs rare, and
anonymity is what let me step in and out of the automation — but they're
infrastructure for a loop that already works. Adopting them first would be
building a scheduler before you have anything worth scheduling.

## Does a stronger model make this unnecessary?

The part most likely to age is the 200k session boundary. The question is
whether stronger models also remove the need for the rest. I don't think they
do, for different reasons.

The distinction from the opening is enough here. A stronger model still cannot
infer a rejected design that left no trace in the artifact. It can re-derive
more conclusions cheaply, so fewer new facts may pass the derivation-cost test.
That retunes future admission; the current protocol does not automatically
re-audit and prune old entries.

The ~200k session boundary will move too, but the workflow is not the number.
Multi-session handoff, fresh-context review, a task file that outlives every
session, and absorption at completion describe where state lives when work
outlasts a conversation. Stronger models make those mechanisms cheaper: more
tasks finish in one development session, so fewer handoffs are needed. Fresh
review still contributes a different position — no memory of writing the code.

I think of the protocol as **a consumer of model capability, not a compensator
for weak models.** A compensator dies when its target deficiency is fixed; a
consumer gets better when its input does.

I would still not extend the claim to every part. Session hooks, output
verification, a scheduler that assembles prompts — those are mechanisms, and
vendors are building them natively; some of this scaffolding should be absorbed
and I would rather it were. What does not get absorbed is the policy: which facts
are worth keeping, what a review may still raise, when a task is done. Those are
judgments about a specific project, and no amount of model progress makes a
project's accumulated decisions organize themselves.

## Why not a general memory layer

There's a healthy research and product line in agent memory — MemGPT/Letta,
Mem0, Cognee, a dozen memory-bank implementations. They solve a more general
problem: give an agent durable recall across sessions, for any domain.

This is deliberately narrower, and the narrowness is the point. **Because it
only has to serve one lifecycle — software work, moving from decision to
implementation to review to closeout — it can hold opinions a general system
cannot assume by default.** Without project-specific policy, a domain-agnostic
layer does not know that _rejected alternatives_ are worth more than function
signatures, that _intentional omissions_ are unusually expensive to recover, or
that a finding set should freeze at first review. Those are not retrieval
improvements. They are claims about what matters in this domain.

I would use one of those for general agent memory. This protocol is for the case
where the domain is known and a strong opinion is more useful than a flexible
one.

## What it cost

These measurements were taken after five months, across four repositories and
**286 completed tasks.**

|                                       |                                              |
| ------------------------------------- | -------------------------------------------- |
| Largest project                       | ~122k lines of Go, 150 tasks                 |
| Sessions per task                     | 3.8 average, development and review combined |
| Full snapshot corpus, largest project | ~34k words, split into sub-documents twice   |

**The flagship project was not greenfield.** Its first commit predates `.ai/` by
ten months; the snapshot was derived from a repository that already had 119 Go
files and its own history.

Sessions are the workflow's operating boundary, not a token or dollar measure.
Most ran on a top-tier model at high reasoning effort. I ask one to wrap around
~200k tokens because precision degrades before capacity is exhausted, though
the heaviest archived session reached ~390k. I use subscriptions rather than
metered API calls, so the archive supports no credible per-task dollar figure.

The workflow also carries a fixed ceremony cost: every development and review
session reloads the snapshot and writes a log, and every task pays for review
and absorption. Those costs amortize only when the project lives long enough
for preserved decisions to be reused. I would not put this on a disposable
three-week project; it would end before the memory had much chance to repay its
overhead.

**For me, it has meant more work per task and less re-derivation over a
project's lifetime.** What I stopped paying is the repeated rediscovery of
things the project already knew — and the occasional catastrophic version where
an agent fails to rediscover one and confidently undoes a deliberate decision.

**And N = 1.** One operator, four repositories, one person's judgment about
what constitutes a good outcome. No A/B test. I do not know how much of what
worked was the protocol and how much was that writing it forced me to think
clearly about my own process. That confound is real and I can't separate it.
Treat every number here as an existence proof, not an effect size.

### What doesn't work

- **Too much ceremony for small projects.** You can run both roles through one
  subscription and put cheaper models on them, but the workflow itself has no
  lightweight mode yet: every task still pays for task tracking, a separate
  review, session-end bookkeeping, and absorption. On a weekend project, that
  lifecycle is absurd.
- **Model identifiers rot.** Profiles name specific models and effort levels;
  they're wrong every few months.
- **Review misses things.** Fresh-context review catches a class of bug the
  author can't see and misses a class the author would have caught. It's a
  different reviewer, not a better one.
- **Absorption is the weakest link.** Everything downstream depends on the
  quality of the distillation at task completion, and that varies with the
  model, the task, and how tired I was reading the result.
- **There is no whole-snapshot size gate.** Closeout checks the documents it
  touches and can leave a housekeeping flag, but nothing continuously scans the
  whole snapshot. One project reached 89% of the derived six-document
  eager-memory bound without a global warning.

## Measuring the protocol itself

Once a protocol is precise enough for a scheduler to execute, it is precise
enough to audit mechanically: illegal transitions, missing handoff fields,
unreviewed work, and failed closeouts all leave detectable evidence. I now
compute a deterministic compliance report beside a model-graded qualitative
one. Whether batched evidence can improve the protocol without letting one
unusual task rewrite policy — or a candidate govern the run that produced it —
is the next question. It needs real batch data before it earns another post.

## Try it

[Mandrel](https://github.com/qinglin89/mandrel) is Apache-2.0. The protocol
documents are the substance; the deployment tool and scheduler are how I run
them, and are the parts most likely to be wrong for your setup. Issues and
discussion are open; I'm not taking code contributions yet.

You do not need to hand an unattended agent your repository to test the idea:

1. **Twenty-minute experiment.** Apply derivation cost, stability, and leverage
   to whatever memory file your agent already reads. Remove anything that
   clearly fails even one test; compress and keep borderline cases.
2. **Full workflow, run manually.** Follow the
   [Quickstart](https://github.com/qinglin89/mandrel#quickstart), then drive
   development and fresh-context review yourself. Interactive use keeps your
   agent's normal permission prompts.
3. **Optional unattended loop.** Try the scheduler only after the manual
   workflow earns its ceremony. It runs agents with filesystem permission
   prompts disabled; it is not a sandbox. Read the
   [safety notes](https://github.com/qinglin89/mandrel#safety), use a repository
   whose worktree you're willing to lose, on a machine you control, and start
   supervised.

The admission-test edge cases are the part I most want tested outside my own
projects. If they fail for yours, I'd like to know why.
