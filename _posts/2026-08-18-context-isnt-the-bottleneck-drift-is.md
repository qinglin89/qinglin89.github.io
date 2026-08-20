---
layout: post
title: "Context isn't the bottleneck. Drift is."
date: 2026-08-18 10:00:00
description: A protocol for long-horizon AI coding, and what 286 tasks across four repositories taught me about it.
og_image: https://qinglin89.github.io/assets/img/projects/mandrel-cover.jpg
tags: ai-coding agents protocol
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

Not "what's in the file" — the agent can read the file. The session needed to
know that the resync design it was about to delete was deliberate and correct.
That a previous task had already ruled on it. That the idempotency angle looks
promising and is a dead end, for reasons that live on the server side. That
three months earlier someone decided this module family owns its own stall
detection, so the fix belongs in the main loop, not in the submit path.

All of it load-bearing, and none of it reachable by reading harder — but for two
different reasons.

Most of it is **decisions**: what was chosen, what was rejected, why. Those are
absent from the artifact. Nobody writes down the design they didn't build, so no
amount of reasoning over the code recovers them, at any model capability.

The idempotency item is a different animal. That one *is* implied by the code —
read the server's idempotency cache, notice it cannot separate "executed" from
"confirmed", and you would get there. It is a **conclusion**, and somebody
already paid for it. Re-deriving it costs a cross-service investigation that no
session working on a twenty-line retry fix is going to fund.

There are three common answers to that question, and I have watched all three
fail.

**Keep the conversation going.** Long threads, resumed sessions, a context
window measured in millions. This works until it doesn't, but not because the
window is full. It fills with the transcript of how you got somewhere while the
conclusions become harder to find. In my runs, capacity has not been the binding
constraint; precision has. Fitting more history into the window did not make
that history more relevant.

**Put it in `CLAUDE.md` / `AGENTS.md` / a memory bank.** Better — now it
survives the session. But every one of these I have seen, including my own
first three attempts, converges to the same end state: a file that only grows,
that nobody deletes from because deleting requires knowing what's still true,
and that eventually costs more to read than it saves. A log is not a memory. A
log with an append-only policy is a log with extra steps.

**Front-load a spec.** The spec-driven frameworks — spec-kit, BMAD, OpenSpec —
are genuinely good at the thing they do, which is turning a vague idea into an
executable plan at the start of a project. They answer "how do I begin?" I
needed an answer to "it's month four, how do I continue?" A plan written in
month one describes a project that no longer exists by month four; the
interesting state is everything learned since, and that is exactly what the
spec doesn't hold.

The failure mode these three share has a name, and it isn't context length.
It's **drift**: decision history accumulating without curation until neither
the human nor the agent can face it. You can feel it happening. Sessions get
longer. The agent re-derives things it derived last week. You start prefacing
every prompt with a paragraph of "remember that we decided…". Eventually you
stop trusting the output and read every diff yourself, which is where the
productivity you bought disappears.

**Drift follows project time more than project size.** A large codebase
generated in a week can carry less drift than a much smaller one iterated over
two years. What accumulates is not code but the sediment of choices made and
reversed.

And it is why a bigger window is not the fix. The second kind of thing the
session needed — the conclusion somebody already paid for — is not a note to be
stored. It is a **starting altitude**: the next session begins reasoning where
the last one stopped, so the depth behind a conclusion keeps accumulating while
the context holding it does not. A larger window does not make re-derivation
free. It only lets you pay for it in one sitting.

So I spent five months building the thing I actually wanted, and running it on
real work. This is what I learned.

---

## The invariant

The invariant is one line:

> **The working set stays bounded over unbounded project time.**

Day 1 and day 300, a fresh session faces the same *shape* of context: a small
constant set of project invariants, one task, and a routed handful of relevant
documents. The corpus behind it grows. The slice loaded into any given session
does not.

I implemented it with three mechanisms: task-scoped growth, selective memory,
and deterministic control outside the model.

Bounded, though, is only half of what matters. The other half is what the
bounded set contains, and it turns out to be two different things that fail the
"just re-derive it" test for two different reasons:

- **Decisions** are *absent from the artifact*. What was rejected, and why; what
  is deliberately not there. No amount of reasoning over the code recovers
  them, because the information was never written into it.
- **Conclusions** are *present but unaffordable*. The code does imply them, but
  reaching them from zero costs more than a session has. These are what
  compound: each one becomes a starting altitude rather than a destination, so
  the next session's reasoning begins a layer deeper than the last one's did.

The first keeps a session from being wrong. The second lets it get somewhere it
could not otherwise reach.

Two directories carry it. **`.ai/`** is the project's curated memory: a
timeless snapshot of how the system currently is, version-controlled, written
only at task completion. **`.ai-tasks/`** is in-flight work: one file per task,
each accumulating a session log that carries state across sessions. A scheduler
I'll call the **orchestrator** runs the loop over one task unattended, or a
human runs the identical loop by hand.

```text
   .ai/  ───────────────┐          ┌──────────────────────────┐
   curated memory       ├─────────►│                          │
   eager set: project   │          │       ONE SESSION        │──► commits
   invariants + routing │          │   fresh context, ~200k   │
   routed: the docs     │          │                          │──► one session-log
   this task named      │          │  works under exactly one │    entry + a status
                        │          │  role contract — dev or  │
   .ai-tasks/<id>.md ───┘          │  review — that names no  │
   goal · scope ·                  │  other role              │
   acceptance ·                    └──────────────────────────┘
   session log                       ▲                    │
                                     │ assembles          │ verifies
                                     │                    ▼
        ┌────────────────────────────┴───────────────────────────────────┐
        │  ORCHESTRATOR — outside the model, holds no flow state         │
        │  re-parses the task file every turn · assembles the prompt ·   │
        │  verifies declared outputs · counts convergence budgets ·      │
        │  escalates to a human on anything it may not decide itself     │
        └────────────────────────────────────────────────────────────────┘

   on completion: absorption distils the session log into .ai/,
   the task file moves to archive, and the loop ends.
```

Everything a session declares goes in the task file. Everything durable it
learned goes into `.ai/`, but only at the end, and only if it passes the tests
in Mechanism 2.

---

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
- **It is where memory comes from.** The snapshot in Mechanism 2 is not written
  from nowhere — absorption reads the accumulated session log at completion and
  distils it. No task, no raw material.
- **It is the audit record.** Status transitions, which session claimed what,
  review verdicts, unresolved disputes, the convergence group a finding belongs
  to. How a change was arrived at survives in a form you can read back, not just
  what the diff ended up being.

The estimate is denominated in a specific unit:

> one session ≈ one effective context window (~200k tokens)

A task estimated `0/3` claims this work needs roughly three windows. If it
turns out to need five, the estimate is raised and the plan re-sliced
mid-flight. The estimate is not a schedule. It's a **sizing constraint**, and
it exists because of an asymmetry I underestimated at the start:

**A task that fits in one session gets completed. A task that doesn't gets
handed off — and every handoff loses information.**

Not "can lose". Loses. The outgoing session knows a hundred things it never
writes down, because it doesn't know which ninety-nine are irrelevant. So the
protocol's job is not to make handoffs lossless — that's unachievable — but to
make them **rare and structured**: rare by sizing work to fit, structured by
specifying exactly what a handoff must carry.

Per session, four fields: **Done** (what happened — committed changes,
decisions made *including rejected alternatives and why*, facts learned worth
recording), **Next** (remaining work), **Open** (unresolved questions), and a
status declaration.

The one that earns its keep is *rejected alternatives*. Here is the actual
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
default. Bounded means *demoted*, not *discarded*, and those are very different
promises to make about a project you intend to work on for a year.

That archive turns out to earn its keep in a second way I did not build it for.
Three months on you are a stranger to your own decisions, and the thing you
actually want then is not a chat log — it is the scoped increment, what it
accepted, what the review said, what stayed unresolved. It is also what makes
delegation survivable: you can hand work to an agent and check afterwards at the
level of decisions rather than reading every diff, which is the only version of
"trust it" that is not just hoping.

---

## Mechanism 2: memory admits, not accumulates

`.ai/` is a small directory describing the project as it currently is.
Timeless: no changelog, no "we used to do X", no dated decisions. Architecture,
design principles and their tradeoffs, module topology, conventions, a routing
index, a feature-to-module map.

The format matters less than the **write policy**, which has two halves.

**First half: writes happen at exactly one moment.** Not during work. A session
that notices `.ai/` is wrong does not fix it — it records the discrepancy in
its session log and moves on. Absorption happens only at task completion, as a
distinct step, reading the task's whole session log at once.

I initially thought this was pointless ceremony. It is now one of the two rules
I'd defend hardest, because **a mid-task session is the worst possible author of
durable knowledge.** Its model of the work is local, partial, and still
changing; what it believes at hour two may be wrong by hour five. Deferring the
write to completion means what gets absorbed is a *finished* understanding,
written by a reader of the whole arc rather than a participant in one leg of
it.

**Second half: a fact must pass three tests to get in.**

| Test | Passes when |
|---|---|
| **Derivation cost** | Re-deriving it requires multi-file traversal, cross-module reasoning, or git archaeology |
| **Stability** | It stays true across iterations without re-verification |
| **Leverage** | Knowing it changes the agent's next action |

All three, or it doesn't enter. Borderline on any one → compress to a single
keyword-dense line and keep it. Don't deliberate; deliberation costs more than
the line.

The exclusions show what those tests are doing:

- Function signatures fail **derivation cost** — grep is cheaper than a doc.
- "We're currently refactoring auth" fails **stability** — true for two weeks.
- "This project uses PostgreSQL" fails **leverage** — the agent discovers it the
  moment it matters, and knowing it early changes nothing.

What passes: invariants, topology, non-obvious couplings, anti-patterns,
vocabulary, **intentional omissions**, runtime dispatch paths that static
reading won't reveal.

*Intentional omissions* need special treatment because they are nearly
impossible to recover from code: absence leaves no trace. An
agent cannot distinguish "there is no cache here because it's deliberate" from
"there is no cache here because nobody got to it," and it will helpfully add
the cache. A snapshot recording the *why* of an absence buys something no
amount of reading can reconstruct.

Here is a real entry, from the design document of the project in the opening
story:

```markdown
### Static loader imports versus dynamic memory entrypoints

- Chose target-aware deploy rendering for one agent surface and dynamic hooks
  for the others.
- Consequence: status separately detects stale/ambiguous entrypoints because
  normalized content hashes intentionally treat legal file forms as equivalent.
```

Two sentences. Re-deriving that consequence means reading the deploy renderer,
the status checker, and two hook scripts, and noticing why a hash comparison
cannot see the thing it needs to see. That is what a passing fact looks like.

### Why this resists degradation

My concern was that repeated distillation would produce a summary of a summary:
a photocopy of a photocopy. A year of that would leave sediment, not knowledge.

The design resists that degradation, and the reason is structural: **each
round re-grounds against the code, not against the previous round's
conclusions.** A session reads `.ai/`, then works in the actual repository —
running tests, reading diffs, watching things fail. When a stale claim
intersects the code a session touches, reading and testing that code can expose
the contradiction, which is then recorded as a fact learned. The snapshot is a
lens on a source of truth that outranks it, not a replacement for it.

`.ai/` is version-controlled for the same reason. When it's wrong, that's a
diff, reviewable like any other.

### Depth accumulates even though space doesn't

A session reasons from two inputs: the code, and a snapshot that is mostly
*previously reasoned conclusions*. So whatever it concludes stands on those —
which means its conclusion can be several inferential layers deep while the
session itself only ever paid for the top layer. Without the snapshot, reaching
that same conclusion from code alone would take a token budget that does not
exist. With it, an equivalent conclusion is reachable in a window that does.

**That is the derivation-cost test, at a higher threshold.** The facts that pass
it most emphatically are exactly the compounded ones: the test that selects them
and the argument for their value turn out to be the same test.

Here is a real two-layer instance. Earlier I quoted this entry:

```markdown
### Static loader imports versus dynamic memory entrypoints
- Consequence: status separately detects stale/ambiguous entrypoints because
  normalized content hashes intentionally treat legal file forms as equivalent.
```

That took real work to establish: it requires knowing why one agent surface gets
statically rendered imports while the others resolve dynamically, and then
noticing that the consequence is a *category* of failure — a file can hash
correctly and still not be the thing that loads.

Months later, workflow skills moved from a user-level directory to per-repository
deployment. A fresh session, with that line in context, asked the next
question: agent tools resolve same-named skills personal-level over
project-level, so does the same category apply? It does. A leftover personal copy
wins over the deployed one — silently, while the manifest, the lockfile, and the
drift check all report agreement, because none of them can see it. That became a
new drift state, `shadowed skill`, and a new snapshot entry.

Nothing in the code says that. The code says a hash matched. Getting from "a
hash matched" to "hashes are structurally blind to which of two legal things is
in force, so any two-location mechanism needs its own check" is the layer that
was already paid for — and the second finding cost one question instead of a
rediscovery.

**Recording a derived conclusion does not retire its source.** There is no
containment hierarchy in knowledge: `1 + 1` is not demoted by calculus being
built on it. Entries leave the snapshot when they stop being true, never because
something was derived from them. A curation policy that prunes ancestors is
cutting the ground out from under its own conclusions.

**And the compounding is not implemented.** There is no mechanism in this system
that grows reasoning depth, no formula that models the expansion. There is an
admission rule, and the depth is what the rule entails. I tried the other way
first — describing the growth directly, so it could be engineered — and it turns
into an endless series of patches, because you are specifying something whose
job is to emerge. The working-set bound has the same shape: nothing computes it;
a size ceiling is enforced and the rest follows.

---

## Inside the task lifecycle: review is a bounded state machine

Development is only one part of a task's lifecycle. Review has to check the
landed work, and the review loop has to terminate.

I tried the simple version — "have another agent review it" — and got an
unbounded loop. Reviewer finds issues. Developer
fixes them. Reviewer reviews the fixes and — being a fresh session with no
memory of what it already decided not to raise — finds a new
set of issues in code it already passed. One task in my archive records six
consecutive changes-requested rounds against six remediation sessions. It
converged, but not because the loop was designed to.

The loop terminates when you add three things.

**1. Convergence groups, frozen at first review.** Every review entry names the
work session anchoring its finding chain. A group's scope is fixed the moment
it opens: *that* finding set, plus regressions introduced by fixing it. A
re-review checks two things only — are the original findings resolved, and did
the fixes break something. It may not raise pre-existing design or style issues
the first review left unflagged. A fix session never opens a new group.

That last clause is load-bearing. Without it, every remediation round is a
fresh comprehensive review of a moving target, and the process is a random walk
rather than a convergence.

**2. Severity gates completion, and only correctness blocks.** Findings are
classified — correctness, design, test, style. Only correctness blocks a task
from completing. A design or test finding is either cheap enough to fix in
place, or it becomes a *new task* while the current review passes. Style never
blocks.

This is a values statement disguised as a rule: a review's job is to stop
broken things from shipping, not to hold work hostage until it's ideal. The
alternative — everything blocks — produces reviewers that are technically never
wrong and processes that never terminate.

**3. Disputes escalate once, immediately.** When a developer session disagreed
with a finding and the reviewer, on the merits, still holds it, the reviewer
records `Dispute-unresolved:` — once — and does **not** re-request the change
in later rounds. It goes to the human.

Two competent parties who disagree after examining the same code have found a
question more rounds won't settle. Looping is the wrong response to genuine
disagreement; adjudication is. Round budgets exist too — two re-reviews per
group, then a binding human ruling — but the dispute path skips the budget and
escalates on round one.

The findings remain a reviewer's judgment. The rules that make the loop
terminate are explicit task state and policy, precise enough for a deterministic
caller to enforce.

---

## Mechanism 3: deterministic control lives outside the model

The task and memory mechanisms define state and policy, but something still has
to execute the loop: choose the next legal role, assemble its context, deliver
its contract, verify its declared outputs, count convergence budgets, and stop
for a human when policy has no mechanical answer.

I built a scheduler that runs this loop headlessly using the configured agent
and model for each role. The model supplies judgment; the scheduler owns the
dispatch and checks that do not require it.

### Caller anonymity keeps execution interchangeable

The temptation, once you have that, is to write the protocol *for the
scheduler*. Role documents saying "the review session will then examine your
work." Prompts restating the rules. State living in the scheduler's memory. I
did all three on the first pass, and the result had a property that made it
worth tearing back out: **it stopped working when I ran a session by hand.**

So the rule became: a session faces only its own contract. Inputs — the task
file, assembled context, role knowledge — its responsibility, its declared
outputs. **It never needs to know that other kinds of session exist.** No role
document names another role. No prose predicts what runs next. The contract is
an anonymous, self-contained work specification; a session knows its own role,
not the caller or what the caller will run next.

**Any datum the scheduler needs from a session is reified as a field with
role-local meaning.** When a remediation session hasn't finished its fix set,
it declares `fix-set: open`. The contract explains that flag in purely local
terms — *declare this when your fix set is incomplete*. Its scheduling
consequence — *the loop dispatches another development session instead of a
review* — is documented only on the scheduler's side. Same datum, two readings,
one owner each. The session isn't told what its declaration will cause, because
a session that knows what its declarations cause starts optimizing for the
outcome rather than reporting the truth.

**The scheduler is a dumb function of the task file.** It holds no flow state.
Every iteration re-parses the file and re-derives what to do next. That means
the loop can be stopped mid-flight, resumed a week later, or — the real prize —
**interleaved arbitrarily with sessions I run by hand.** Same runbook, two
executors. I can drive it manually with a slash command that delivers the same
contract text the scheduler would have injected, byte-equivalently, and the
machine picks up where I left off without noticing anything happened.

That property is worth more than any individual rule above. Automation you
can't step into is automation you must trust completely or abandon completely.
This one you can take the wheel on at any session boundary.

To keep it honest rather than aspirational, the boundary rules are checked
mechanically: four litmus tests — does a role document name another role? does
any prose predict what runs next? is a field documented by its consequence
instead of its local meaning? does a prompt restate a rule instead of
instantiating it? — and a lint script that catches the greppable ones on every
change. Principles that aren't checked decay. These are checked.

### The enforcement ladder

The protocol is enforced through five layers. They differ in the dimension that
matters here: **whether a model can decline to comply.**

| Layer | Where it runs | What it catches |
|---|---|---|
| **Protocol docs** — conduct, schemas, project invariants | ambient in the model's context | a decision made without the invariants in view |
| **Role contract** — dev or review, delivered at invocation | ambient, but the only imperative text a session gets | doing a different job than the one asked for |
| **Skills** — packaged procedures (intake, closeout, housekeeping) | the model invokes them | a multi-step procedure done from memory and half-right |
| **Hooks** — session start and stop, in the agent's own runtime | **outside the model** | ending on a dirty tree; a session-log entry written before the work landed |
| **Orchestrator** — prompt assembly, output verification, budgets | **outside the model** | a session that reports work it did not declare, or a loop that will not converge |

The top three are semantic: they work by being read and understood, which
means they work most of the time. The bottom two are mechanical: they run
outside the model's judgment and cannot be reasoned with. A stop hook does not
care how good the explanation for the dirty tree is.

That split is the design. The semantic layers carry everything that requires
judgment, because only a model can supply it. The mechanical layers carry the
handful of properties that must hold *regardless* of judgment — the tree is
clean, the entry exists, the declared output is really there — and they are
deliberately few, because every mechanical check is a rule you can no longer
change by argument.

This is as close as I can get to making an agent follow a protocol. I cannot
guarantee semantic compliance, so the parts that matter most do not depend on
it.

---

## Did the invariant actually hold?

The claim is that the working set stays bounded while project time doesn't. That
was measurable, but I had never measured it. Since `.ai/` is version-controlled
and every archived task records the documents it pulled in, I could reconstruct
it retroactively: for each of 150 completed tasks in the project from the
opening story, in order, how many bytes did a fresh session load before it could
start?

Here, "bounded" does not mean constant. Assembly has two parts. The **eager
set** is what every session pays
regardless of task: the routing index, the map, and the current entrypoint of
four core documents. The **routed** part is what that particular task named in
its frontmatter.

Only the eager set could grow with project age, and the protocol puts a hard
ceiling on it: a single document may not exceed ~3000 tokens and the index
~1500, so six eagerly-loaded documents cap out around **16k tokens**. When a
document outgrows its limit it is split, and the overflow moves to the routed
tier where only tasks that need it pay. That ceiling is the claim. The question
is whether it holds.

Here is the eager set, sampled across four months:

```text
task    date         eager set (bytes)
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

It's a sawtooth, and it stays under the ceiling. It oscillates between roughly
4k and 10k tokens — against a cap of 16k — and the correction fired twice.
Today it sits at about half its ceiling, four months and 150
tasks in. The number of eagerly-loaded documents stayed constant at six the
whole time; only their contents moved.

The sawtooth is more useful to me than a flat line would have been: it shows the
correction firing twice.

The routed term behaves differently, and should: it's a function of the task,
not of the project's age. A task touching thirteen subsystems names thirteen
documents, on day 1 or day 300. Total assembly — eager plus routed — averaged
~5k tokens across the first fifteen tasks and ~15k across the last fifteen, and
the single heaviest task in the whole set assembled ~43k, about a fifth of the
200k budget. Growth, then, but growth in the term that tracks how wide a task
is rather than how old the project is.

Then I ran the same measurement on a second project, 87 tasks, and it is the
more useful result. Its eager set corrected once, early, and has climbed
monotonically since — to ~14k tokens, **89% of the ceiling, with no split.**
The mechanism has not fired there when it should have. That snapshot is overdue
for exactly the correction the first project took twice, and the housekeeping
that would do it is manual and I had not run it.

So: a ceiling that holds where it's enforced, and one live case of it not being
enforced. Writing the measurement script exposed the lapse: the invariant is
only as good as the thing that checks it, and that check was a habit rather than
a gate.

---

## Which part is load-bearing

If you take one thing, take the **admission tests**. Derivation cost,
stability, leverage. They're twenty minutes of work, they need none of the rest
of this, and they convert a memory file from a log into something with an
editorial policy. Apply them to whatever file your agent already reads and
delete everything that fails all three.

If you take two, add **fresh-context review**. A separate session with no
memory of writing the code is well positioned to catch defects the author
misses. In my runs, even cheaper reviewers did so; fresh context contributes
something capability alone does not.

**Task sizing and role anonymity are the ones you start paying for after the
first two are running.** They matter — sizing is what keeps handoffs rare, and
anonymity is what let me step in and out of the automation — but they're
infrastructure for a loop that already works. Adopting them first would be
building a scheduler before you have anything worth scheduling.

---

## Does a stronger model make this unnecessary?

The part most likely to age is the 200k session boundary. The question is
whether stronger models also remove the need for the rest. I don't think they
do, for different reasons.

Start with the two kinds of thing memory holds. Decisions are
*immune* to capability: the design you rejected left no trace in the artifact, so
there is nothing there for a better reader to find. Conclusions are *amplified*
by it: a stronger model reaches further in one session, so what gets recorded
starts higher, and the next session starts from there. The derivation-cost test
even retunes itself — it is defined against what re-deriving actually costs, so
as models improve, facts that were expensive stop qualifying and quietly leave.
The snapshot gets smaller, not obsolete.

The task workflow ages differently. Its 200k-token session size is pegged to
current capability and will move. The workflow is not the number. Multi-session
handoff, a separate reviewer, a task file that outlives every session that
touched it, absorption at completion: none of those describe a model's limits.
They describe where state lives when work outlasts a conversation.

What capability changes is how often they cost anything. A stronger model
finishes more tasks in one session, which means fewer handoffs, which means less
lost between them — the mechanism stays and its price falls. Handoff overhead
approaches zero rather than approaching obsolescence, and it is still there on
the day a task is large enough to need it. Fresh-context review is also not only
a capability question: a reviewer brings a different position — no memory of
writing the thing. Using a stronger model for both roles preserves that gap,
just as two stronger engineers do.

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

---

## Why not a general memory layer

There's a healthy research and product line in agent memory — MemGPT/Letta,
Mem0, Cognee, a dozen memory-bank implementations. They solve a more general
problem: give an agent durable recall across sessions, for any domain.

This is deliberately narrower, and the narrowness is the point. **Because it
only has to serve one lifecycle — software work, moving from decision to
implementation to review to closeout — it can hold opinions a general system
cannot.** A general memory layer cannot know that *rejected alternatives* are
worth more than function signatures, that *intentional omissions* are the
category most expensive to recover, or that a finding set should freeze at
first review. Those aren't retrieval improvements. They're claims about what
matters in this domain, and a system that must serve every domain can't make
them.

I would use one of those for general agent memory. This protocol is for the case
where the domain is known and a strong opinion is more useful than a flexible
one.

---

## What it cost

These measurements were taken after five months, across four repositories and
**286 completed tasks.**

| | |
|---|---|
| Largest project | ~122k lines of Go, 150 tasks |
| Sessions per task | 3.8 average, development and review combined |
| Snapshot size, largest project | ~34k words, split into sub-documents twice |

**The flagship project was not greenfield.** Its first commit predates its `.ai/` by
ten months; the snapshot was derived from a repository that already had 119 Go
files and its own history. The protocol was introduced to a codebase in
progress, so these numbers include brownfield adoption rather than only
greenfield setup.

The underlying consumption is tokens. I report sessions because that is where
the workflow draws an operating boundary. Tasks averaged 3.8 sessions,
development and review combined, most at a top-tier model on high reasoning
effort. I ask a session to wrap at roughly 200k tokens of context. That is a
handoff policy, not a hard stop — the heaviest single session in the archive
peaked near 390k. I use the earlier boundary deliberately because precision
degrades before the available context is exhausted.

Each session is one bounded working envelope: it reloads context, occupies a
rate-limit window, and — after the first — introduces a handoff. The 3.8 average
only says how many such envelopes a task used. It does not measure token
consumption; sessions can end well before the policy cap or overrun it.

I run these agents through subscriptions rather than metered API calls, so the
archive does not support a credible per-task dollar estimate. An additional
session has no itemized price, but it still consumes subscription capacity and
wall-clock time. These records describe operating load and turnaround, but they
do not convert the run into a dollar cost.

The workflow also carries a fixed ceremony cost: every session reloads the
snapshot and writes a log, and every task pays for review and absorption. Those
costs amortize only when the project lives long enough for preserved decisions
to be reused. I would not put this on a disposable three-week project; it would
end before the memory had much chance to repay its setup and maintenance.

**For me, it has meant more work per task and less re-derivation over a
project's lifetime.** That is the tradeoff I'd defend hardest. What I stopped
paying is the re-derivation tax — the twenty minutes at the start of every
session where an agent rediscovers something the project already knew, plus the
occasional catastrophic version where it doesn't rediscover it and confidently
undoes a deliberate decision.

**And N = 1.** One operator, four repositories, one person's judgment about
what constitutes a good outcome. No A/B test. I do not know how much of what
worked was the protocol and how much was that writing a protocol forced me to
think clearly about my own process for five months. That confound is real and I
can't separate it. Treat every number here as an existence proof, not an effect
size.

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
- **The size ceiling is enforced by habit, not by a gate.** One snapshot has sat
  at 89% of it without splitting, and nothing told me. Everything else in this
  system that matters is checked mechanically; this isn't, and it should be.

---

## Measuring the protocol itself

Once a protocol is precise enough for a scheduler to execute, it's precise
enough to **measure whether it was followed** — illegal status transitions,
missing handoff fields, unreviewed work, failed closeouts are all mechanically
detectable. I now compute a deterministic compliance report per completed task,
alongside a model-graded qualitative one.

The next question I am working on is whether batched compliance evidence can
improve the protocol itself — without letting one unusual task rewrite policy,
and without letting a candidate revision govern the run that produced it.

I think so, and most of the machinery is built and running. That's a later
post, and it needs real batch data behind it rather than a design document.

---

## Try it

[github.com/qinglin89/mandrel](https://github.com/qinglin89/mandrel) —
Apache-2.0. The protocol documents are the substance; the deployment tool and
scheduler are how I run them, and are the parts most likely to be wrong for your
setup. Issues and discussion are open; I'm not taking code contributions yet.

The smallest useful experiment is the three admission tests. It takes about
twenty minutes and requires adopting nothing else. Their edge cases are the
part I most want tested outside my own projects, so if they fail for yours, I'd
like to know why.
