# Loop Engineering (LE)

> **A loop is only as good as its stopping condition. Terminate on verification you do not control, or do not loop at all.**

LE is a heuristic framework for designing agentic loops and deciding when they should stop.
It codifies the emerging practice of "loop engineering": the shift from writing one-shot
prompts to designing the control loop that drives an agent [1]. It is **grounded in** (not
derived from) two lineages: the agent-loop research literature, where explicit reasoning traces [6],
interleaved reasoning and acting, self-reflection, and iterative refinement are studied
directly [2,3,4], and the older tradition of feedback-driven decision cycles such as the
OODA loop and closed-loop control [5]. LE translates that body of work into a set of practical,
testable rules for two loop families: **execution loops** (plan -> act -> observe ->
reflect) and **refinement loops** (generate -> critique -> revise).

**Status:** Partially grounded, not validated end to end. Unlike a pure hypothesis, several
claims here rest on published results and shipped-tool defaults (iteration caps, cost
limits, the diminishing-returns curve of refinement). But the central design rule and the
specific cap values have not been validated as a combined methodology on a controlled
benchmark. Treat this as a well-sourced starting point for experimentation, not a proven
recipe. The most important claim it makes is contested in the literature (see below), and
LE takes a side while showing the disagreement.

## Epistemic Honesty Notice

Three honesty flags belong up front:

1. **The core claim is contested.** LE asserts that loops should terminate on external
   verification rather than self-judgment. The strongest evidence for this is the finding
   that intrinsic self-correction (no external feedback) can degrade reasoning accuracy
   [10], and a survey concluding self-correction works reliably only with external feedback
   or large-scale fine-tuning [11]. There is a credible opposing position: reinforcement
   learning can train genuine self-correction into a model [12], and reflection-based
   methods do improve results when paired with a signal [4]. LE resolves this by grounding
   termination in external verifiers wherever they exist, and treating pure self-critique
   as a last resort. This is a design choice under uncertainty, not a settled fact.

2. **Many tool specifics are vendor-published and time-stamped.** The loop behaviors,
   iteration defaults, and long-horizon run claims for specific coding agents come from
   engineering blogs and documentation [16,17,18,14], not peer review. Product details
   drift fast. Where a number comes from a shipped tool, it is labeled as such and dated.

3. **The cap numbers are defaults, not constants.** "Refinement 3-4 iterations" and
   "agentic steps ~15-25" are reasonable starting points from cited tools and studies
   [3,13,14], not values derived from first principles. They are domain-dependent and must
   be tuned. The framework's value is the control structure, not the specific integers.

## Why This Exists

Agentic coding tools now run in loops by default: gather context, take an action, verify the
result, repeat [16]. The practice around them has moved accordingly. As the head of one
major coding agent put it in 2026, the job is shifting from prompting a model to writing the
loops that prompt it [1]. That shift creates a new failure surface. Loops that lack a
reliable stop signal do one of two bad things: they quit too early, or they never stop.

The specific failure modes are well documented. Agents repeat a failing action (a doom
loop). They oscillate between two states without net progress. Their reliability degrades as
accumulated context fills with dead ends, well before the context window is full [15]. They
"pass" a goal by gaming the verifier: deleting tests, hardcoding outputs, silencing a type
checker [18]. And on reasoning tasks, a loop that trusts its own judgment about correctness
can make the answer worse with each pass, not better [10].

What separates an agentic project that ships from one that spins is rarely the model. It is
the loop design: a verifiable goal, an external stop signal, sane caps, checkpoints, and a
rule for when to decompose instead of iterate. LE is a checklist for getting that design
right.

## The Framework

LE structures any loop into three parts: a one-time FRAME, the repeating LOOP, and a
one-time CLOSE.

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│   GOAL (from the user)                                       │
│       │                                                      │
│       ▼                                                      │
│   ┌────────────────────┐                                     │
│   │  FRAME (run once)  │  Loop at all? single output -> no   │
│   │                    │  Write a VERIFIABLE goal            │
│   │                    │  Name the VERIFIER                  │
│   │                    │  Set caps + budget                  │
│   └────────┬───────────┘                                     │
│            │                                                 │
│            ▼                                                 │
│   ┌────────────────────┐<-- re-inject the literal goal       │
│   │  LOOP (1..N)       │                                     │
│   │                    │    1. re-inject goal                │
│   │  per iteration:    │    2. act (one step / one revision) │
│   │                    ├--> 3. observe: EXTERNAL feedback    │
│   │                    │    4. terminate?                    │
│   │                    │       verifier pass -> SUCCESS      │
│   │                    │       no progress   -> STOP         │
│   │                    │       budget hit    -> ESCALATE     │
│   │                    │    5. failure signal? -> BREAK,     │
│   │                    │       roll back, decompose          │
│   └────────┬───────────┘                                     │
│            │                                                 │
│            ▼                                                 │
│   ┌────────────────────┐                                     │
│   │  CLOSE (run once)  │  Report outcome + confidence        │
│   │                    │  Flag residual risk                 │
│   │                    │  Checkpoint or escalate             │
│   └────────┬───────────┘                                     │
│            │                                                 │
│            ▼                                                 │
│   OUTPUT                                                     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Formal Specification

```
FUNCTION loop_engineer(goal: str, env: Environment) -> (outcome, confidence, residual_risk):

    // FRAME (run once)
    IF is_single_pass(goal):
        RETURN (do_once(goal), confidence, note_unchecked_risks())

    verifier      := select_verifier(goal, env)   // tests / compiler / types / lint / evaluator
    verifiable    := make_goal_checkable(goal, verifier)
    max_iters     := N            // default: refinement 3-4, agentic 15-25; tunable
    budget        := set_budget() // tokens / dollars / wall-clock
    max_fails     := 3            // consecutive-failure circuit breaker

    prev_state    := known_good_checkpoint(env)
    prev_action   := NULL
    fail_streak   := 0

    // LOOP (1..N iterations)
    FOR i IN 1..max_iters:

        // Step 1: Re-inject the literal goal (prevents drift)
        current_goal := verifiable   // NOT a paraphrase

        // Step 2: Act -- exactly one step or one revision
        action_i := plan_one_step(current_goal, env)   // or: revise_once(...)
        apply(action_i, env)

        // Step 3: Observe with EXTERNAL feedback
        signal := verifier.run(env)   // real result, not self-assessment

        // Step 4: Termination checks, in order of reliability
        IF signal.passed:
            RETURN close(env, i, "success")
        IF separate_evaluator_says_done(goal, env):
            RETURN close(env, i, "success_by_evaluator")
        IF no_progress(signal, action_i, prev_action, prev_state, env):
            RETURN close(prev_state, i, "converged_or_stalled")
        IF over_budget(budget):
            RETURN close(prev_state, i, "budget_exhausted -> escalate")

        // Step 5: Failure-signal checks -> break the loop
        IF detect_loop_failure(action_i, signal, history):   // doom loop, oscillation,
            rollback(env, prev_state)                         // context rot, reward hack,
            RETURN close(prev_state, i, "failure_signal -> decompose")  // drift, etc.

        // Bookkeeping
        IF signal.regressed: fail_streak += 1 ELSE fail_streak := 0
        IF fail_streak >= max_fails:
            rollback(env, prev_state)
            RETURN close(prev_state, i, "circuit_breaker_tripped")
        IF signal.improved:
            prev_state := checkpoint(env)   // commit the good state
        prev_action := action_i

    // Hit the cap without success
    RETURN close(prev_state, max_iters, "cap_reached -> decompose, do not add passes")
```

### Stage 1: FRAME

Run once, before the first iteration. Skipping FRAME is the most common cause of runaway
loops.

| Situation | Loop? | Action |
|---|---|---|
| Deterministic, single correct output | No | Do it once; a loop only adds risk |
| One artifact to improve, verifier exists | Refinement loop | Iterate until verifier passes or cap hit |
| Multi-step task needing tools + environment | Execution loop | Full plan/act/observe/reflect with subgoals |
| One artifact to improve, no external verifier | Loop with caution | Cap hard at 1-2 passes; self-critique is unreliable |

Then: **write a verifiable goal** (checkable by something other than the agent's opinion),
**name the verifier** (test suite, compiler, type checker, linter, runtime check, or a
separate evaluator model), and **set caps and budget** (max iterations, token or dollar
ceiling, consecutive-failure limit). Caps are the backstop, not the intended exit.

**The verifiability problem:** some goals are genuinely hard to verify mechanically (open
design questions, subjective quality). This is exactly where loops are most dangerous,
because there is no reliable stop signal. When no external verifier exists, the honest move
is to shorten the loop drastically, lean on a separate evaluator or human check, and report
the resulting uncertainty rather than iterating on self-assessment until it "feels done."

### Stage 2: LOOP

Each iteration runs five steps. For execution loops the middle is plan/act/observe; for
refinement loops it is generate/critique/revise. The control logic is identical.

**Step 1: Re-inject the goal.** Re-read the literal goal before acting. The agent-loop
literature interleaves reasoning with grounding in the actual task state to keep the agent
from wandering [2]; re-grounding on the original goal each iteration is the cheapest defense
against goal drift.

**Step 2: Act.** Take exactly one concrete step or produce one revision. One step per
iteration keeps observation and rollback clean and preserves attribution when something
breaks.

**Step 3: Observe with external feedback.** Run the verifier and read the real signal: test
results, compiler output, type errors, lint output, a runtime error, or a separate
evaluator's judgment. The central rule of this framework: **the stop/continue decision is
made by a verifier the agent does not control, never by the generator's own sense that it is
finished.** This is the design consequence of the self-correction evidence [10,11].

**Step 4: Check termination,** in order of reliability: (1) verifier passes; (2) a separate
evaluator judges the goal met (maker-checker separation, so the writer is not the grader);
(3) no progress (unchanged output, recurring error, identical recommended action); (4)
budget exhausted (the backstop).

**Step 5: Check failure signals** (below). If one fires, break, roll back to the last
known-good state, and decompose or escalate. Do not loop harder.

**Hitting the cap:** the task probably needs decomposition into smaller verifiable
sub-goals, not more iterations. Restructure; do not add passes.

### Stage 3: CLOSE

After the loop exits: state the outcome plainly (succeeded / partial / failed, against which
verifier); report calibrated confidence, or say it is uncertain rather than inventing a
number; flag residual risk, specifically what the verifier did not check (passing tests do
not mean the tests exercise the new behavior); and checkpoint the good state, or on failure
hand back a clear summary of what was tried and where it stalled so a human or a fresh loop
can resume.

## Iteration Caps

Defaults, not derived constants. What matters is that a cap exists and that the primary stop
signal is a verifier, not the cap.

| Loop type | Default cap | Rationale | Source of the number |
|---|---|---|---|
| Refinement (critique-revise) | 3-4 iterations | Gains concentrate in the first 1-2 passes; quality plateaus by 3-5 and can regress at 5 | Self-Refine and follow-on studies [3] |
| Agentic steps (scoped task) | ~15-25 steps | Successful solves cluster early; a common framework default is 25 | SWE-agent median success near 12 steps [13]; LangGraph default 25 [14] |
| Long-horizon build | Higher, but gated | Only safe with checkpoints, durable progress files, and a hard budget | Long-running-agent harness practice [16] |
| Consecutive failures | 2-3, then trip | Repeating a failing action is the doom-loop signature | Circuit-breaker practice |
| Spend / tokens | Explicit ceiling | Accumulated context makes loops grow costly; cost is roughly quadratic in context | SWE-agent default near $3 per instance [13] |

If a scoped task is not converging near its median success step count, more steps rarely
rescue it. Raise caps only after adding checkpointing and external verification, never to
paper over their absence.

## Loop Failure Signals

Specific failure modes where continuing the loop makes things worse. Each has a test and an
action. These are heuristic indicators, not formal detectors, and they are not exhaustive.

**Signal 1: Doom loop.** Same action, same error, repeatedly.
Test: is this iteration's action and error identical to a previous one?
Action: break, roll back, decompose or change strategy.

**Signal 2: Oscillation.** Undo-redo of the same change (A -> B -> A -> B), or a metric
flipping without net improvement.
Test: does state or the chosen action reverse direction across consecutive iterations?
Action: stop; take the last verifier-passing state.

**Signal 3: No progress.** Iterations run but produce no new passing state or reduced error.
Test: did this iteration change the verifier result or error set at all?
Action: stop; escalate or decompose.

**Signal 4: Context rot.** Reliability degrades as context fills with dead ends, well before
the window is full [15], compounding the known tendency of models to lose information in the
middle of long contexts [20].
Test: is the context long and cluttered with failed attempts, and is quality drifting down?
Action: compact; summarize progress to a durable file; restart with a fresh context or
sub-agent carrying only the goal and known-good state.

**Signal 5: Error accumulation.** Each iteration builds on an unverified prior mistake.
Test: is current work standing on an earlier output that was never verified?
Action: roll back to the last verified state; rebuild from there.

**Signal 6: Reward hacking / specification gaming.** The loop "passes" by weakening the
verifier: deleting or loosening tests, hardcoding outputs, silencing the type checker,
terminating early [18]. Reward hacking is a known general phenomenon in optimizing systems [19].
Test: did the pass come from making the code correct, or the check easier? Does a passing
test actually exercise the new behavior?
Action: reject the pass; re-run against the original, unmodified verifier. A gamed green is
worse than a red, because it hides the failure.

**Signal 7: Overconfident self-correction.** The agent "fixes" correct code and introduces a
new fault, typical when relying on self-judgment [10].
Test: did an external verifier flag the thing being changed, or is this a hunch?
Action: require an external signal before mutating working code.

**Signal 8: Goal drift.** The loop now solves an adjacent but different problem.
Test: does the current work satisfy the literal, re-injected goal?
Action: roll back to the last on-goal iteration; re-inject and resume.

**Signal 9: Hallucination amplification.** A fabricated early claim is treated as
established and reused downstream.
Test: is a claim being relied on that was never grounded in a tool result, file, or source?
Action: stop and verify externally before it propagates.

### Recovery Protocol

Do NOT loop your way out of a failing loop.

1. Break the loop.
2. Roll back to the last known-good, verified state (commit, checkpoint).
3. Re-inject the original goal.
4. Decide: decompose into smaller verifiable sub-goals, change strategy, or escalate with a
   clear summary of what was tried and where it stalled.
5. If you resume, carry only the goal and the known-good state into a fresh context, not the
   full history of failed attempts.

## Goal-Oriented Engineering

How the goal is framed and the loop is steered decides the outcome more than model choice
does.

**Make success externally verifiable.** Deterministic criteria (tests passed, exit codes,
score thresholds) give the loop a reliable stop signal [16]. A goal you cannot verify is a
goal the loop cannot safely end on.

**Decompose; work one unit at a time.** A single loop rarely one-shots a large system. Split
into subgoals, each with its own verifier and small loop, and commit after each working
increment so you can always revert to a passing state [16].

**Ground the loop in external verification over self-critique.** Compilers, tests, linters,
and type checkers are reliable; the agent's own judgment about its reasoning is not,
especially on non-verifiable tasks [10,11].

**Keep durable memory.** A markdown spec or progress file the loop re-reads each iteration,
plus per-increment commits, holds the goal across many iterations and enables recovery after
a context reset [16].

**Separate the maker from the checker.** A reviewer with fresh context, or a separate
evaluator model, catches what the writer is blind to.

**Use scripts for deterministic steps.** Mechanical work (formatting, building, running
tests) does not need a reasoning step; offloading it saves budget and removes a source of
drift.

## Anti-Patterns

**Looping without a verifier.** Iterating on pure self-assessment frequently degrades
reasoning answers rather than improving them [10]. With no external check, cap at 1-2 passes
and be honest about the uncertainty.

**Treating the cap as the stop signal.** The cap is a backstop against runaway loops. If
loops routinely exit by hitting the cap, the goal is not verifiable enough.

**Trusting proxy signals as completion.** A file existing is not the feature working. Tests
passing is not tests that exercise the new behavior. A clean type check is not meaningful
types [18].

**Looping harder when stuck.** More iterations on a stalled loop burn budget and accumulate
noise; a stall is a signal to decompose, not to add passes.

**Batching risky steps per iteration.** Multiple unverified changes in one iteration destroy
attribution and clean rollback.

**No rollback point.** A long loop with no checkpoints lets one bad iteration poison
everything after it.

## When NOT to Use LE

LE adds overhead and is not appropriate for every task.

Do not use LE when: the task is a single deterministic pass with one correct output; the
work is pure open-ended generation with no notion of "correct" (some creative drafting);
the process is already governed by an external protocol (a fixed pipeline, a regulatory
procedure); or latency and budget make even a short loop impractical and a single best-effort
pass is preferable.

Use LE when: the task takes multiple steps with tool use and environment interaction; an
artifact must be improved and a verifier exists; you are seeing runaway loops, doom loops, or
oscillation; or a project's agentic runs fail in ways that trace back to missing stop signals
and missing checkpoints.

## LE Failure Modes

LE itself can fail. These are the framework's own failure modes, distinct from the loop
failure signals it detects.

| Failure mode | Description | Est. frequency | Impact | Mitigation |
|---|---|---|---|---|
| No verifier available | The goal has no mechanical check, so LE's core rule cannot apply | Medium | Loop falls back to self-judgment, the weak case | Use a separate evaluator or human check; shorten the loop; report uncertainty |
| Verifier is gameable | The agent passes by weakening the check (reward hacking) | Medium | False success that hides a real failure | Signal 6 detection; re-run against an unmodified verifier; hold tests immutable |
| Wrong cap for the domain | Default cap too low (quits early) or too high (burns budget) | Medium | Missed solutions or wasted compute | Treat caps as tunable; log per-goal step and cost counts to recalibrate |
| Premature termination | No-progress or convergence fires while the answer is still wrong | Low-Medium | Incorrect result delivered as done | Prefer verifier-based exit over convergence; add an adversarial check |
| Self-monitoring failure | The agent cannot reliably detect its own failure signals | High | All heuristic detectors weaker than assumed | External monitoring (logging, human review of flagged runs) is the real mitigation |
| Framework overhead | FRAME and per-iteration checks cost tokens without improving outcome on easy tasks | Medium | Wasted compute | "When NOT to Use" guidance; skip LE for single-pass tasks |

The most important entry is self-monitoring failure. Several failure signals ask the agent
to notice its own malfunction, which is asking the system to grade itself. As with the
self-correction evidence, this is inherently limited [10,11]. In production, external
monitoring, logging iteration counts and costs, and human review of flagged runs are the
only reliable mitigations.

## Cost Model

Each iteration costs tokens, latency, and, for tool-using loops, real actions. Two cost
properties matter. First, accumulated context grows every iteration, and because each step
re-sends prior context, total cost tends to grow faster than linearly with iteration count
[17]. Second, degradation is not free either: a loop that runs past the point of diminishing
returns pays for iterations that do not improve, and can pay again to undo damage from
overcorrection [3,10].

Break-even intuition: LE's overhead (FRAME plus per-iteration checks) is justified when it
prevents runaway loops and premature stops that would otherwise cost more in wasted
iterations or wrong answers. On single-pass tasks, LE is pure overhead and should be skipped.
This break-even has not been measured on a controlled benchmark.

## Validation Methodology

LE is not validated end to end. The following would test it.

**Test 1: Cap calibration.** On a benchmark with known-solvable tasks, sweep the iteration
cap and measure solve rate and cost. Expected: solve rate saturates, so a cap near the
saturation point maximizes solve-per-cost. Automatable on coding benchmarks.

**Test 2: Verifier-based vs self-judgment termination.** Compare loops that stop on external
verifiers against loops that stop on the agent's own judgment, on the same tasks. Hypothesis
from the literature: verifier-based termination yields higher correctness and fewer
overcorrection regressions [10,11]. This is the central claim and the definitive test.

**Test 3: Failure-signal detection.** Construct runs designed to trigger each signal (doom
loop, oscillation, reward hacking). Measure detection rate and false-alarm rate. Expected:
detection is imperfect and needs the backstop of hard caps.

**Test 4: End-to-end quality and cost.** Compare LE-governed loops against ungoverned loops
on a diverse agentic benchmark for solve rate, token cost, and wall-clock. Without this, LE
remains a structured hypothesis.

**Test 5: Domain calibration.** Repeat on distinct domains (code fixing, multi-file feature
work, data pipelines). Caps and signals may need per-domain tuning.

## Assumptions

Every assumption in this framework, with status and disposition.

| # | Assumption | Status | Disposition |
|---|---|---|---|
| 1 | External verification is a more reliable stop signal than self-judgment | Moderate | Supported by [10,11]; opposed by [4,12]; framework takes this side explicitly |
| 2 | A verifier exists or can be constructed for the goal | Weak | Often false for open-ended goals; "When NOT to Use" and fallback added |
| 3 | Refinement gains concentrate in early iterations | Moderate | Supported by [3]; magnitude is domain-dependent |
| 4 | Default caps (3-4 refinement, 15-25 agentic) transfer across domains | Unverified | Flagged as tunable, drawn from [3,13,14], not derived |
| 5 | Agents can self-detect loop failure signals | Weak | Acknowledged as limited; external monitoring is the real mitigation |
| 6 | Re-injecting the goal each iteration reduces drift | Moderate | Plausible by analogy to [2]; unproven as an isolated intervention |
| 7 | One step per iteration improves attribution and rollback | Moderate | Engineering judgment; not separately benchmarked |
| 8 | Checkpointing enables safe long-horizon loops | Moderate | Supported by harness practice [16]; implementation-dependent |
| 9 | LE improves outcomes over ungoverned loops | Unverified | No end-to-end validation; methodology provided |
| 10 | Framework is model-agnostic | Moderate | The loop shape and verifier rule are portable; tool specifics differ |
| 11 | Loop failure signals are exhaustive | Weak | Explicitly stated as non-exhaustive |

## Calibration Heuristics

Starting-point cap and verifier choices. Surface-level heuristics, not true calibration
(which needs per-domain feedback). They will be wrong in specific cases.

| Signal | Default | Rationale | Known exceptions |
|---|---|---|---|
| "Fix the failing tests" | Refinement loop, cap 3-4, stop on green | Tests are the verifier | Flaky tests need stabilizing first |
| "Rename / mechanical edit" | No loop, single pass | Deterministic | Large refactors may need staged verification |
| "Build feature X" | Execution loop, decompose, commit per unit | Compositional, long-horizon | Trivial features may be single-pass |
| "Improve this prose / design" | Cap 1-2, use a separate evaluator or human | Often no mechanical verifier | If a rubric exists, treat it as the verifier |
| "Optimize until metric >= T" | Loop, stop at threshold, watch for gaming | Threshold is the verifier | Guard against reward hacking (Signal 6) |
| Same error twice | Break (doom loop) | Repetition signals a dead end | None |
| Context long and cluttered | Compact, fresh context | Context rot [15] | None |
| No external verifier at all | Shorten drastically, report uncertainty | Self-judgment is the weak case | None |

## Integration With Other Frameworks

LE governs when loops stop and how they are steered. It composes with the methods that run
inside a loop.

| Framework | Integration point | Note |
|---|---|---|
| ReAct [2] | ReAct is the inner execution cycle; LE governs its caps and stop conditions | LE adds explicit termination and failure-signal detection |
| Self-Refine [3] | Self-Refine is a refinement loop; LE bounds its iterations and prefers an external verifier over self-feedback | LE constrains the plateau/regression risk |
| Reflexion [4] | Reflexion's reflection feeds the next iteration; LE requires that reflection be checked against an external signal | Complementary; LE hardens the stop rule |
| Tree of Thoughts [8] / LATS [9] | Branching search inside a step; LE's caps and budget bound tree expansion | LE governs total cost, not per-branch logic |
| Adaptive Depth Reasoning [21] | ADR calibrates depth within a single reasoning step; LE calibrates iteration across steps | Sibling skills: ADR = how deep per step, LE = how many steps and when to stop |
| Self-consistency [7] / Best-of-N | An alternative to iterative refinement: sample N independently and select | LE helps decide between best-of-N (parallel, no error accumulation) and refinement (feedback, but compounding risk) |

## Installation

### Claude

**Claude.ai (web/mobile/desktop):** Navigate to Settings > Customize > Skills, click the
"+" button, then "+ Create skill." Upload the `loop-engineering/` folder (containing
`SKILL.md` and `theory.md`). Toggle the skill on. Requires Code execution to be enabled in
Settings > Capabilities.

**Claude Code (terminal agent):** Copy the skill folder to your personal skills directory.
The skill activates automatically when Claude Code detects a matching task, or invoke it
explicitly with `/loop-engineering`.

```
# Personal install (available across all projects)
cp -r loop-engineering/ ~/.claude/skills/loop-engineering/

# Project install (available only in this repo, shared via git)
cp -r loop-engineering/ .claude/skills/loop-engineering/
```

**Claude API:** Copy the content of `SKILL.md` (without the YAML frontmatter between `---`
markers) into the `system` parameter of your API request.

### Gemini CLI

Gemini CLI uses the same SKILL.md format. Copy the skill folder to one of Gemini's skill
discovery directories. Gemini auto-discovers skills at session start and activates them via
the `activate_skill` tool when a task matches the description.

```
# User-level (available across all workspaces)
cp -r loop-engineering/ ~/.gemini/skills/loop-engineering/
# OR use the cross-agent alias path:
cp -r loop-engineering/ ~/.agents/skills/loop-engineering/

# Workspace-level (available only in this project)
cp -r loop-engineering/ .gemini/skills/loop-engineering/
```

Verify installation with `/skills list` inside a Gemini CLI session. The skill should appear
as `loop-engineering`.

**Gemini API / Google AI Studio:** Copy the content of `SKILL.md` (without the YAML
frontmatter) into the `system_instruction` parameter.

### OpenAI Codex

Codex CLI, the VS Code extension, and the Codex desktop app all share the same skill
discovery. Codex scans `.agents/skills/` directories from your current working directory up
to the repository root. Skills use the same SKILL.md format.

```
# Repository-level (shared via git, available to all Codex users of this repo)
cp -r loop-engineering/ .agents/skills/loop-engineering/

# User-level (available across all repos)
# Add this path to your ~/.codex/config.toml if not already configured
cp -r loop-engineering/ ~/.codex/skills/loop-engineering/
```

Invoke with `/skills` or `$loop-engineering` in the Codex CLI, or let Codex activate it
implicitly when a task matches the description.

**ChatGPT (web):** Copy the content of `SKILL.md` (without YAML frontmatter) into Settings >
Personalization > Custom Instructions, or create a custom GPT with the content as its
instruction set.

**OpenAI API:** Include the content of `SKILL.md` (without YAML frontmatter) as a `system`
message in your messages array.

### Cross-Agent Compatibility

The SKILL.md format is an emerging open standard. The same skill folder works across Claude
Code, Gemini CLI, and Codex CLI without modification. For environments that share a single
skill directory across agents, the `~/.agents/skills/` path is recognized by both Gemini CLI
and Codex CLI natively, and Claude Code can be pointed at it via the `--add-dir` flag.

```
# Shared cross-agent installation
cp -r loop-engineering/ ~/.agents/skills/loop-engineering/
```

### Any Other LLM

Copy the content of `SKILL.md` (without the YAML frontmatter between `---` markers) into your
model's system prompt or custom instructions. The framework is plain text and requires only
that the model can take actions in a loop and observe external feedback.

## Quick Reference

```
FRAME (once):
  1. Loop at all? single deterministic output -> no loop
  2. Write a VERIFIABLE goal (checkable by something other than your opinion)
  3. Name the VERIFIER (tests / compiler / types / lint / evaluator)
  4. Set caps: max iters, token/$ budget, max_consecutive_failures = 2-3

LOOP (defaults: refinement 3-4, agentic steps ~15-25):
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
Cap reached without success -> decompose, do not add passes.
```

## Project Structure

```
loop-engineering/
├── README.md         # This file
├── SKILL.md          # Skill definition (usable as system prompt)
└── theory.md         # Research foundations and citations
```

## Acknowledgments

This framework builds on the agent-loop research community, in particular the work on
interleaved reasoning and acting [2], iterative self-refinement [3], verbal
self-reflection [4], and tree-structured agent search [8,9]. It takes its position on
termination from the critical literature on LLM self-correction [10,11] while acknowledging
the opposing reinforcement-learning results [12]. It draws its concrete defaults and loop
patterns from the engineering practice around current coding agents [16,17,18] and framework
documentation [14], and its framing of "loop engineering" as a discipline from recent
practitioner writing [1]. It is designed as a sibling to the Adaptive Depth Reasoning
skill [21]: where that skill governs how deeply to reason within a step, this one governs how
many steps to take and when to stop.

For the full research treatment, including each architecture, the self-correction debate, the
diminishing-returns evidence, the documented failure modes, and how current tools implement
loops, see [`theory.md`](theory.md).

## References

[1] A. Osmani, "Loop Engineering: The Guide for AI Agents," 2026. Consolidating practitioner writing on the shift from prompting to loop design; popularized in remarks by B. Cherny (Anthropic) and framing by A. Karpathy and P. Steinberger.

[2] S. Yao, J. Zhao, D. Yu, N. Du, I. Shafran, K. Narasimhan, and Y. Cao, "ReAct: Synergizing reasoning and acting in language models," arXiv preprint arXiv:2210.03629, 2022 (ICLR 2023).

[3] A. Madaan et al., "Self-Refine: Iterative refinement with self-feedback," arXiv preprint arXiv:2303.17651, 2023 (NeurIPS 2023).

[4] N. Shinn, F. Cassano, A. Gopinath, K. Narasimhan, and S. Yao, "Reflexion: Language agents with verbal reinforcement learning," arXiv preprint arXiv:2303.11366, 2023 (NeurIPS 2023).

[5] F. P. B. Osinga, "Science, Strategy and War: The Strategic Theory of John Boyd," Routledge, 2007. Primary secondary source for the OODA loop.

[6] J. Wei, X. Wang, D. Schuurmans, M. Bosma, B. Ichter, F. Xia, E. Chi, Q. Le, and D. Zhou, "Chain-of-thought prompting elicits reasoning in large language models," arXiv preprint arXiv:2201.11903, 2022 (NeurIPS 2022).

[7] X. Wang et al., "Self-consistency improves chain of thought reasoning in language models," arXiv preprint arXiv:2203.11171, 2022.

[8] S. Yao, D. Yu, J. Zhao, I. Shafran, T. L. Griffiths, Y. Cao, and K. Narasimhan, "Tree of Thoughts: Deliberate problem solving with large language models," arXiv preprint arXiv:2305.10601, 2023 (NeurIPS 2023).

[9] A. Zhou et al., "Language Agent Tree Search unifies reasoning, acting, and planning in language models," arXiv preprint arXiv:2310.04406, 2023 (ICML 2024).

[10] J. Huang, X. Chen, S. Mishra, H. S. Zheng, A. W. Yu, X. Song, and D. Zhou, "Large language models cannot self-correct reasoning yet," arXiv preprint arXiv:2310.01798, 2023 (ICLR 2024).

[11] R. Kamoi, Y. Zhang, N. Zhang, J. Han, and R. Zhang, "When can LLMs actually correct their own mistakes? A critical survey of self-correction of LLMs," Transactions of the Association for Computational Linguistics, vol. 12, pp. 1417-1440, 2024 (arXiv:2406.01297).

[12] A. Kumar et al., "Training language models to self-correct via reinforcement learning," arXiv preprint arXiv:2409.12917, 2024 (ICLR 2025).

[13] J. Yang, C. E. Jimenez, A. Wettig, K. Lieret, S. Yao, K. Narasimhan, and O. Press, "SWE-agent: Agent-computer interfaces enable automated software engineering," arXiv preprint arXiv:2405.15793, 2024 (NeurIPS 2024).

[14] LangChain, "GRAPH_RECURSION_LIMIT," LangGraph documentation, 2025. Default recursion limit of 25.

[15] K. Hong, A. Troynikov, and J. Huber, "Context Rot: How increasing input tokens impacts LLM performance," Chroma Research Technical Report, July 2025.

[16] Anthropic, "Effective harnesses for long-running agents," Anthropic Engineering, 2025.

[17] Anthropic, "Getting started with loops," Claude blog, June 30, 2026.

[18] OpenAI, "Unrolling the Codex agent loop," OpenAI, 2025.

[19] V. Krakovna et al., "Specification gaming: The flip side of AI ingenuity," DeepMind Safety Research Blog, 2020.

[20] N. F. Liu, K. Lin, J. Hewitt, A. Paranjape, M. Bevilacqua, F. Petroni, and P. Liang, "Lost in the middle: How language models use long contexts," arXiv preprint arXiv:2307.03172, 2023.

[21] op7ic, "Adaptive Depth Reasoning Skill," GitHub repository, 2026. Sibling skill and structural template for this repository. [Online]. Available: https://github.com/op7ic/Adaptive_Depth_Reasoning_Skill
