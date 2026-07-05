---
name: loop-engineering
description: >
  A practical framework for designing and controlling agentic loops. Use this skill
  whenever an agent will run more than one step to reach a goal: plan/act/observe/reflect
  execution loops, or generate/critique/revise refinement loops. Trigger when the task
  involves deciding HOW MANY times to iterate, WHEN to stop, or HOW to avoid infinite
  loops, doom loops, oscillation, context rot, or reward hacking. Also trigger when the
  user mentions "loop engineering", "agentic loop", "agent loop", "iteration cap",
  "iteration limit", "termination condition", "stopping condition", "self-correction",
  "critique and revise", "reflection loop", "goal-oriented engineering", "when to stop
  iterating", "loop until tests pass", or "keep it from looping forever". This skill is
  model-agnostic and works with any tool-using or reasoning LLM (Claude, Codex, Gemini,
  and others).
---

# Loop Engineering

A practical framework for designing the loops that agents run, and for deciding when those
loops should stop. The core principle:

**An agent should not be the sole judge of when its own loop is done.** Terminate on
verification you do not control (tests, compilers, type checkers, linters, or a separate
evaluator), detect no-progress early, and cap the rest. Loops grounded in self-judgment
plateau, and on reasoning tasks they often degrade.

This framework codifies the emerging practice of "loop engineering": the shift from writing
one-shot prompts to designing the control loop that drives an agent. It is grounded in the
agent-loop research literature (ReAct, Reflexion, Self-Refine, LATS) and in the older
lineage of feedback-driven decision cycles (the OODA loop, closed-loop control). It covers
the two loop families the practice is built on:

- **Execution loops** -- plan -> act -> observe -> reflect, over multiple tool-using steps.
- **Refinement loops** -- generate -> critique -> revise, iterating on a single artifact.

Both share the same skeleton and the same failure modes. What separates a project that ships
from one that spins forever is almost never the model. It is the loop design: a verifiable
goal, an external stop signal, sane caps, checkpoints, and a rule for when to decompose
instead of iterate.

## The Loop, In One Picture

```
  FRAME (once)                LOOP (1..N iterations)              CLOSE (once)
  ┌───────────────┐           ┌────────────────────────┐         ┌───────────────┐
  │ define a      │           │ 1. re-inject the goal  │         │ report result │
  │ VERIFIABLE    │──────────>│ 2. act (one step)      │────────>│ + confidence  │
  │ goal          │           │ 3. observe: EXTERNAL   │         │ + residual    │
  │ set caps +    │  <──loop──│    feedback            │         │   risk        │
  │ budget        │           │ 4. check termination   │         │ checkpoint or │
  │ confirm a     │           │ 5. check failure signal│         │ escalate      │
  │ VERIFIER      │           │    -> break/rollback   │         │               │
  └───────────────┘           └────────────────────────┘         └───────────────┘
```

## Stage 1: FRAME (before you loop)

Do this once, before the first iteration. Skipping it is the most common cause of runaway
loops.

**1. Decide whether to loop at all.**

| Situation | Loop? | Action |
|---|---|---|
| Deterministic, single-pass, one obvious correct output | No | Do it once. A loop adds only risk. |
| One artifact to improve, and a verifier exists (tests, types, lint) | Refinement loop | Iterate until the verifier passes or the cap is hit. |
| Multi-step task needing tool use and environment interaction | Execution loop | Full plan/act/observe/reflect with subgoals. |
| One artifact to improve, and NO external verifier exists | Loop with caution | Cap hard at 1-2 passes. Pure self-critique is unreliable (see below). |

**2. Write a verifiable goal.** The goal must be checkable by something other than the
agent's opinion. Not "fix the bugs" but "every test in `/tests/unit/` passes with exit
code 0, and no files are created outside `/src/`." A goal you cannot verify is a goal you
cannot safely loop toward.

**3. Confirm the verifier.** Name the thing that will say "done": a test suite, a compiler,
a type checker, a linter, a runtime smoke check, or a separate evaluator model. If you
cannot name one, treat the loop as high-risk and shorten it drastically.

**4. Set the budget and caps.** Before starting, fix: max iterations, a token or dollar
budget, and a consecutive-failure limit. These are the backstop, not the primary stop
signal. Defaults are in the table below.

## Stage 2: LOOP (the execution cycle)

Each iteration runs these five steps in order. For execution loops the middle is
plan/act/observe; for refinement loops it is generate/critique/revise. The control logic is
identical.

**Step 1: Re-inject the goal.** Re-read the original, literal goal before acting. Not your
paraphrase of it, not the sub-task you drifted into. Long loops wander; re-grounding on the
actual goal every iteration is the cheapest defense against goal drift.

**Step 2: Act.** Take exactly one concrete step, or produce one revision. One step per
iteration keeps observation and rollback clean. Do not batch three risky edits into one
iteration; you lose the ability to attribute which one broke things.

**Step 3: Observe with EXTERNAL feedback.** Run the verifier. Get the real signal: test
results, compiler output, type errors, lint output, a runtime error, or a separate
evaluator's judgment. Do not substitute your own confidence for this. The single most
important design rule in this skill: **the stop/continue decision is made by a verifier the
agent does not control, never by the generator's own sense that it is finished.**

**Step 4: Check termination.** In order of reliability:

1. **Verifier passes** (tests green, exit 0, compiler clean, types clean). Stop: success.
2. **Separate evaluator judges the goal met** (a different model or a checklist scoring the
   result). Stop: success. This is maker-checker separation; the writer is not the grader.
3. **No progress** (the output is unchanged, or the same error recurs, or the recommended
   action is identical to last iteration). Stop: you have converged or stalled.
4. **Budget exhausted** (iteration cap, token/dollar cap, or wall-clock hit). Stop: escalate.

**Step 5: Check failure signals.** Before looping again, scan for the loop failure signals
below. If one fires, break the loop, roll back to the last known-good state, and either
decompose or escalate. Do not "loop harder."

**When you hit the cap without success:** the task probably needs decomposition into smaller
sub-goals, each with its own verifier, not more iterations. Restructure the problem. Adding
passes past the cap is how token budgets get burned for nothing.

## Stage 3: CLOSE (after the loop)

Once the loop exits:

1. State the outcome plainly: succeeded, partially succeeded, or failed, and against which
   verifier.
2. Report confidence as a calibrated estimate. If you cannot estimate it, say so rather than
   inventing a number.
3. Flag residual risk: what the verifier did NOT check (passing tests do not mean the tests
   exercise the new behavior), and what could still be wrong.
4. Checkpoint the good state (commit, save a progress file). On failure, hand back a clear
   summary of what was tried and where it stalled, so a human or a fresh loop can resume.

---

## Iteration Caps That Make Sense

These are defaults, not derived constants. They are starting points drawn from real tools
and studies, and they must be tuned per domain. What matters more than the exact number is
that a cap exists and that the primary stop signal is a verifier, not the cap.

| Loop type | Default cap | Why | Source of the number |
|---|---|---|---|
| Refinement (critique-revise) | 3-4 iterations | Most gains land in the first 1-2 passes; quality often plateaus by 3-5 and can regress at 5 | Self-Refine and follow-on refinement studies |
| Agentic steps, scoped task | ~15-25 steps | Successful solves cluster early; a common framework default is 25 | SWE-agent median success near 12 steps; LangGraph default recursion limit 25 |
| Long-horizon build | Higher, but gated | Only safe with checkpoints, durable progress files, and a hard budget | Long-running-agent harness practice |
| Consecutive failures (circuit breaker) | 2-3 then trip | Repeating a failing action is the doom-loop signature | Common circuit-breaker practice |
| Spend / tokens | Set an explicit ceiling | Loops are quadratic in accumulated context; cost grows fast | SWE-agent default near $3 per instance |

Rules of thumb:

- If a scoped task is not converging by roughly the median success step count, more steps
  rarely rescue it. Stop and decompose.
- More refinement iterations are not more rigor. Past 3-4 with no external signal, you are
  usually accumulating noise, not quality.
- Raise caps only when you have added checkpointing and external verification, not to
  compensate for their absence.

## Loop Failure Signals -- BREAK if you detect these

These are the loop analog of overthinking. Each is a specific failure mode where continuing
the loop makes things worse. Each has a test you can apply and an action to take.

**Signal 1: Doom loop (repeated failed action).** The same action is retried and produces
the same error.
Test: is this iteration's action and error identical to a previous one?
Action: break. Roll back. The environment is telling you this approach does not work.
Decompose or change strategy.

**Signal 2: Oscillation.** The agent undoes and redoes the same change (A -> B -> A -> B),
or a metric flips back and forth without net improvement.
Test: does state or the chosen action reverse direction across consecutive iterations?
Action: stop. You lack a discriminating signal. Report the tension; pick the last
verifier-passing state.

**Signal 3: No progress.** Iterations run but produce no new passing state, no new insight,
no reduced error count.
Test: did this iteration change the verifier result or the error set at all?
Action: stop. Looping is not moving the goal. Escalate or decompose.

**Signal 4: Context rot.** Over a long loop, reliability degrades as accumulated context
fills with distractors, dead ends, and stale output. This happens well before the context
window is full.
Test: is the context long and cluttered with prior failed attempts, and is quality
drifting down?
Action: compact. Summarize progress into a durable file, start a fresh context (or a
fresh sub-agent) carrying only the goal and the known-good state.

**Signal 5: Error accumulation.** Each iteration builds on the last iteration's mistake, so
errors compound instead of resolving.
Test: is the current work standing on an earlier output you never verified?
Action: roll back to the last verified state and rebuild from there, not from the corrupted
intermediate.

**Signal 6: Reward hacking / specification gaming.** The loop "passes" by cheating the
verifier: deleting or weakening tests, hardcoding expected outputs, silencing the type
checker (`as any`, blanket ignores), or terminating early to score a pass.
Test: did the pass come from making the code correct, or from making the check easier? Does
a passing test actually exercise the new behavior?
Action: reject the pass. A green check that was gamed is worse than a red one, because it
hides the failure. Re-run against the original, unmodified verifier.

**Signal 7: Overconfident self-correction.** The agent "fixes" something that was already
correct and introduces a new fault. Common when the loop relies on self-judgment rather than
an external signal.
Test: did an external verifier actually flag the thing being changed, or is this a change on
a hunch?
Action: require an external signal before mutating working code. If none flagged it, do not
touch it.

**Signal 8: Goal drift.** The loop is now solving an adjacent but different problem than the
one asked.
Test: does the current work satisfy the literal, re-injected goal from Step 1?
Action: roll back to the last iteration that addressed the actual goal. Re-inject and
resume.

**Signal 9: Hallucination amplification.** A fabricated fact from an early iteration gets
treated as established and reused downstream.
Test: is a claim being relied on that was never grounded in a tool result, file, or source?
Action: stop and verify the claim externally before it propagates further.

## Recovery Protocol

Do NOT try to loop your way out of a failing loop. That is more of the same failure.

1. Break the loop.
2. Roll back to the last known-good, verified state (git commit, saved checkpoint).
3. Re-inject the original goal.
4. Decide: decompose into smaller verifiable sub-goals, change strategy, or escalate to a
   human with a clear summary of what was tried and where it stalled.
5. If you resume, resume with a fresh context carrying only the goal and the known-good
   state, not the full history of failed attempts.

---

## Goal-Oriented Engineering: How You Direct the Loop Decides the Outcome

The gap between an agentic project that ships and one that fails is mostly in how the goal
is framed and how the loop is steered. Concrete practices:

**Make success externally verifiable.** If a test, compiler, or evaluator cannot confirm
"done," the loop has no reliable stop signal and will either quit too early or grind
forever. Prefer deterministic criteria (tests passed, exit codes, score thresholds).

**Decompose; work one unit at a time.** A single loop rarely one-shots a large system.
Split into subgoals, each with its own verifier and its own small loop. Commit after each
working increment so you can always revert to a state that passed.

**Ground the loop in external verification over self-critique.** Compilers, tests, linters,
type checkers, and runtime checks are reliable signals. The agent's own judgment about
whether its reasoning is correct is not, especially on non-verifiable tasks. Design the loop
so the verifier, not the generator, decides.

**Keep durable memory.** Long loops lose the thread. A markdown spec or progress file that
the loop re-reads each iteration, plus per-increment commits, is the most reliable way to
hold a goal across many iterations and to recover after a context reset.

**Separate the maker from the checker.** A reviewer with fresh context, or a separate
evaluator model, catches what the writer is blind to. The writer should not be the sole
grader of its own work.

**Use scripts for deterministic steps.** If part of the loop is mechanical (formatting,
building, running tests), call a script. Deterministic work does not need a reasoning step,
and offloading it saves budget and removes a source of drift.

## Anti-Patterns

**Looping without a verifier.** Iterating on pure self-assessment. On reasoning tasks this
frequently degrades the answer rather than improving it. If there is no external check, cap
at 1-2 passes and be honest about the uncertainty.

**Treating the iteration cap as the stop signal.** The cap is a backstop against runaway
loops, not the intended exit. If your loops routinely exit by hitting the cap, your goal is
not verifiable enough.

**Trusting proxy signals as completion.** A file existing is not the feature working. Tests
passing is not tests that exercise the new behavior. A clean type check is not meaningful
types. Confirm the proxy actually implies the goal.

**Looping harder when stuck.** More iterations on a stalled loop burn budget and accumulate
noise. A stall is a signal to decompose or change strategy, not to add passes.

**Batching risky steps per iteration.** Multiple unverified changes in one iteration destroy
attribution and clean rollback. One step, one observation.

**No rollback point.** Running a long loop with no checkpoints means one bad iteration can
poison everything after it with no way back. Commit working states.

## Quick Reference

```
FRAME (once):
  1. Loop at all? deterministic single output -> no loop
  2. Write a VERIFIABLE goal (checkable by something other than your opinion)
  3. Name the VERIFIER (tests / compiler / types / lint / evaluator)
  4. Set caps: max iters, token/$ budget, max_consecutive_failures = 2-3

LOOP (default caps: refinement 3-4, agentic steps ~15-25):
  a. Re-inject the goal (literal, not paraphrased)
  b. Act: exactly one step / one revision
  c. Observe: EXTERNAL feedback (run the verifier)
  d. Terminate?  verifier pass -> success
                 evaluator says done -> success
                 no progress / same error -> stop
                 budget hit -> escalate
  e. Failure signal? doom loop / oscillation / context rot / reward hack /
                     goal drift -> BREAK, roll back, decompose

CLOSE (once):
  - Outcome + confidence + residual risk (what the verifier did NOT check)
  - Checkpoint the good state; on failure, hand off a clear summary

RULE: a verifier you do not control decides "done", not the generator.
If you hit the cap without success -> decompose, do not add passes.
```

---
