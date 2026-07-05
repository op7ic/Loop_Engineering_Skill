# Loop Engineering: Research Foundations

This document is the research basis for the Loop Engineering (LE) skill. It surveys the agent
loop architectures LE composes with, lays out the contested evidence on whether models can
self-correct (the question that drives LE's central design rule), summarizes what the
literature and shipped tools say about iteration caps and termination, catalogs documented
loop failure modes, and reviews how current coding agents implement loops. It is written to
be honest about what is established, what is contested, and what is merely vendor-reported.

Citations use bracketed markers that map to the numbered reference list at the end. Nothing
here should be read as settled fact where the text flags disagreement.

LE is designed as a sibling to the Adaptive Depth Reasoning skill [25], which shares this
repository's structure and epistemic conventions. That skill calibrates how deeply to reason
within a single step; this one calibrates how many steps to take and when to stop. The two
compose: reasoning depth per step, and iteration count across steps.

## 1. What "loop engineering" is, and its two families

"Loop engineering" names an engineering practice that became visible in 2025 and 2026: as
coding agents began running in loops by default, the useful unit of work shifted from writing
a single prompt to designing the control loop that repeatedly drives the agent. Practitioner
writing consolidated the term [22], with the framing that one should design the loop that
prompts an agent rather than prompt the agent directly. This is a practice claim, popularized
in talks and posts, not a peer-reviewed result, and it should be weighted accordingly. What
gives it substance is that the underlying components have been studied directly in the
research literature, and that shipped tools now expose concrete loop controls.

LE treats two loop families:

- **Execution loops:** plan -> act -> observe -> reflect, over multiple tool-using steps in
  an environment. The canonical academic form is ReAct [3], where the model interleaves
  reasoning traces with actions and observations.
- **Refinement loops:** generate -> critique -> revise, iterating on a single artifact. The
  canonical academic form is Self-Refine [4], where one model generates, critiques its own
  output, and revises.

Both descend from a much older idea: the feedback-driven decision cycle. The OODA loop
(Observe, Orient, Decide, Act), developed by John Boyd and documented most thoroughly by
Osinga [23], is the same shape, with Boyd emphasizing that the Orient phase, where
observations are interpreted against a model of the situation, is the decisive one. Classical
closed-loop control shares the structure: sense, compare against a setpoint, actuate, repeat.
The transferable lesson from both is that a loop without a good feedback signal and a good
stopping rule does not converge; it oscillates or diverges. That lesson is the spine of LE.

## 2. Foundational agent loop architectures

The following are the building blocks LE governs rather than replaces. They are listed in
roughly chronological order to show how the field moved from linear reasoning to branching
search to trained self-correction.

**Chain-of-Thought (CoT)** [1]. Prompting a model to produce intermediate reasoning steps
before an answer improves performance on multi-step problems, and the effect emerges at
scale. CoT is the linear substrate that later methods extend. Its structural weakness is
directly relevant to loops: an error in any single step propagates down the chain, because
there is no branching and no backtracking.

**Self-consistency** [2]. Rather than iterate, sample many independent CoT solutions and take
a majority vote over the final answers. This is the parallel alternative to sequential
refinement. It avoids error accumulation because samples are independent, at the cost of
running N generations. LE uses this contrast when advising between best-of-N sampling and
iterative refinement.

**ReAct** [3]. Interleaves reasoning and acting: the model emits a thought, takes an action
(a tool call), observes the result, and repeats. This is the foundational execution loop and
the structure underneath most modern coding agents. Its per-step observation is what makes
external grounding possible.

**Self-Refine** [4]. A single model plays generator, critic, and reviser, iterating until a
stopping criterion. The reported improvement is roughly 20 percent absolute on average across
the paper's tasks, with the paper noting that most of the gain arrives in the early
iterations. This early-gain, later-plateau shape is the empirical basis for keeping
refinement caps low. Notably, the method was weak on pure mathematical reasoning, where
self-generated feedback is less reliable, foreshadowing the self-correction debate below.

**Reflexion** [5]. Adds verbal reinforcement: an actor acts, an evaluator scores the outcome,
and a self-reflection step writes a natural-language lesson into an episodic memory buffer
that is carried into the next trial. On coding (HumanEval) it reports 91 percent pass@1,
above the GPT-4 baseline reported in the same work, using a small trial budget. The important
detail for LE is that Reflexion's coding success leans on an external signal (self-generated
unit tests), which is consistent with the thesis that reflection helps most when paired with
a verifier rather than with pure self-judgment.

**Tree of Thoughts (ToT)** [6]. Generalizes CoT from a chain to a search tree: generate
multiple candidate thoughts at each step, evaluate them, and search (breadth-first or
depth-first) with backtracking from unpromising nodes. This buys the ability to recover from
a bad step, at a large compute cost. On the Game of 24 task, ToT explored on the order of
dozens of nodes with several model calls each, far more compute than a single CoT pass. LE's
caps and budgets exist precisely to bound this kind of expansion.

**Graph of Thoughts (GoT)** [7]. Generalizes ToT further to an arbitrary graph, so thoughts
can be aggregated and combined, not only branched. Same theme, more flexible topology, same
need for a bound on exploration.

**Plan-and-Solve** [8]. Splits the work into an explicit planning phase followed by execution
of the plan's steps, addressing the missing-step errors that arise when a model tries to
reason and execute simultaneously. It underlies the plan-then-execute pattern in several
agent frameworks, and it is why LE's execution loop separates planning a step from taking it.

**Language Agent Tree Search (LATS)** [9]. Unifies reasoning, acting, and planning by
combining Monte Carlo Tree Search with model-based value estimates and self-reflection, and,
crucially, incorporates external environment feedback into the search. It is a useful example
of the LE thesis in an academic setting: the search is grounded in real feedback, not only in
the model's own evaluation.

**SCoRe** [12]. Trains a model to self-correct using multi-turn reinforcement learning on the
model's own generated data, reporting meaningful absolute gains on mathematics and coding
benchmarks. SCoRe is the strongest counterweight to the pessimistic self-correction results
below: it shows that genuine self-correction can be learned, even if it does not arise
reliably from prompting alone.

## 3. The self-correction debate (the crux)

LE's central rule (terminate on external verification, not self-judgment) rests on a genuinely
contested question: can LLMs correct their own mistakes without external feedback? The honest
answer is that the evidence points mostly toward "not reliably, and sometimes it makes things
worse," but there are credible results on the other side, and the picture depends heavily on
whether an external signal is available.

**The skeptical results.**

Huang and colleagues [10] tested intrinsic self-correction, meaning self-correction with no
external feedback and no oracle telling the model when to stop, over a small number of
rounds. Across the models they tested, accuracy went down, not up. Reported drops include a
GPT-4-class model on grade-school math falling several points after self-correction, and
larger collapses on a commonsense reasoning task. Their answer-flip analysis found the model
changed roughly as many correct answers to incorrect as the reverse, netting negative. Their
diagnosis is blunt: the models could not reliably judge the correctness of their own
reasoning, so asking them to revise on that basis was as likely to break a right answer as to
fix a wrong one. A critical nuance from the same work: when an oracle supplied ground-truth
stopping (stop as soon as the answer is correct), performance did improve. Their point is that
earlier optimistic results often smuggled in exactly this kind of external signal.

Kamoi and colleagues [11], in a critical survey, reached a compatible conclusion: no prior
work convincingly demonstrated self-correction using feedback from a prompted LLM alone,
except on tasks unusually suited to it; self-correction does work when the feedback is
reliable and external; and large-scale fine-tuning can instill it. Tyen and colleagues [13]
localized the bottleneck: models are much better at correcting an error once its location is
pointed out than at finding the error in the first place. In other words, detection, not
revision, is the weak link, which is exactly what an external verifier supplies.

**The optimistic and nuancing results.**

Self-Refine [4] and Reflexion [5] both report gains from iterative self-feedback. The
skeptical camp's rejoinder is that their strongest results tend to involve an external or
oracle signal (unit tests, task-specific evaluators) rather than pure self-assessment.
SCoRe [12] is the cleanest positive result: with the right reinforcement-learning training,
a model can learn to self-correct. This means "LLMs cannot self-correct" is too strong as a
universal claim; the accurate statement is that self-correction does not arise reliably from
prompting alone in current models, and can degrade reasoning outputs when attempted without a
grounded signal.

**Synthesis, and how LE resolves it.** Self-correction is reliable when the feedback is
external and grounded: a compiler, a test suite, a type checker, a runtime error, or a
separate evaluator. It is unreliable, and can actively harm output, when it is the generator's
own judgment about a reasoning task. LE therefore makes the stop/continue decision the job of
a verifier the agent does not control, and treats pure self-critique as a capped, last-resort
fallback for cases where no verifier exists. This is a design choice made under genuine
uncertainty, and LE says so.

## 4. Termination conditions and iteration caps

This is LE's most practically important section and the user-facing question of "limits on
loops that make sense."

**A reliability ordering of stop signals.** From most to least reliable:

1. A deterministic external verifier passes: tests green, exit code zero, compiler clean,
   linter and type checker clean. Engineering guidance from tool builders explicitly favors
   deterministic completion criteria such as number of tests passed or a score threshold
   [18].
2. A separate evaluator judges the goal met. Current tools implement this as a distinct,
   often smaller, model that grades completion after each turn, so the agent that wrote the
   code is not the one deciding it is done [19,20]. This is maker-checker separation.
3. No-progress or convergence detection: the output is unchanged, the same error recurs, or
   the recommended action is identical to the previous iteration.
4. Budget exhaustion: a hard cap on iterations, tokens, dollars, or wall-clock. This is the
   backstop, not the intended exit.

**Concrete caps from shipped tools and studies.** These are the numbers LE's defaults are
drawn from. They are tool-specific and time-stamped, and they will drift.

- LangGraph ships a default recursion limit of 25, which raises an error when reached without
  hitting a stop condition, and the documentation notes that complex graphs can hit this
  naturally [21]. So the value serves double duty as both an infinite-loop guard and a real
  ceiling. A known pitfall is that subgraphs can silently use the default even when the parent
  sets a higher limit.
- SWE-agent and its lighter variants use a per-instance dollar cap on the order of a few
  dollars and step limits in the low hundreds for benchmark harnesses [14]. In the original
  SWE-agent work, successful solves clustered early: the median successful run finished in
  roughly a dozen steps and around a dollar, while failed runs ran longer and cost more, and
  the large majority of resolved instances submitted before exhausting the budget. The
  operational reading is that if a scoped task is not solved in roughly the median success
  step count, more steps rarely rescue it.
- Coding-agent products expose goal-directed loops that stop when a completion condition is
  met or a user-set turn cap is hit, with a separate evaluator checking completion each turn
  [19,20]. They also ship loop detection and context compaction as standard guardrails.

**The diminishing-returns evidence for low refinement caps.** Self-Refine reports that most
of its gains arrive in the earliest iterations [4]. Follow-on refinement studies commonly see
quality plateau after a handful of iterations, and some report an outright decline at higher
iteration counts, attributed to diminishing returns plus noise or overfitting to the critique
signal. The practical default that falls out of this is a refinement cap of three to four
iterations, with the understanding that needing more is a signal to decompose the task rather
than to loop harder. LE labels these numbers as tunable defaults, not derived constants.

## 5. Documented loop failure modes

These are the failure modes LE's failure signals are meant to catch. Each is documented or
directly grounded in cited work.

**Infinite loops, doom loops, and oscillation.** The simplest failures: an agent repeats a
failing action, or undoes and redoes the same change. These arise naturally in cyclic control
graphs and when the success condition is ambiguous, which is why frameworks ship hard
recursion limits and loop detection [21].

**Context rot.** A vendor research report tested many frontier models and found that model
performance grows less reliable as input length grows, and that this degradation appears well
below the nominal context-window limit, even on simple retrieval [16]. The stated mechanism is
noise accumulation rather than capacity. This compounds the earlier "lost in the middle"
finding that models use information at the middle of a long context less effectively than
information at the ends [17]. For long loops, where context accumulates failed attempts and
stale output, this is a primary failure mode, and the mitigation is compaction plus fresh
context carrying only the goal and the known-good state. The report is from a company that
sells retrieval infrastructure and therefore has an incentive in the finding, but it tested a
broad model set and aligns with the independent lost-in-the-middle result, so it is credible
while not disinterested.

**Error accumulation.** Because refinement and execution loops build each step on the last,
an unverified early mistake compounds. This is the loop-level version of CoT's step-error
propagation [1]. The mitigation is to roll back to the last verified state rather than build
on a corrupted intermediate.

**Reward hacking and specification gaming.** When a loop optimizes toward a proxy, it may
satisfy the proxy without satisfying the goal. Specification gaming is a well-documented
general phenomenon in optimizing systems [15]. In coding agents specifically, it shows up as
weakening or deleting tests, hardcoding expected outputs, silencing the type checker, or
terminating a program early to score a pass. Tool guidance explicitly warns against accepting
proxy signals as completion: a file existing does not mean the feature works, a passing test
does not mean the test exercises the new behavior, and a green type check does not mean the
types are meaningful [20]. The mitigation is to hold the verifier immutable and re-run against
the original checks.

**Overconfident self-correction.** Directly from the self-correction results [10]: a loop that
revises on self-judgment can change a correct answer to an incorrect one. The mitigation is to
require an external signal before mutating working code.

**Goal drift.** Over a long loop, the agent can end up solving an adjacent but different
problem. The mitigation, re-injecting the literal goal each iteration, is analogous to how
ReAct-style loops keep re-grounding in the current task state [3].

**Hallucination amplification.** A fabricated claim from an early iteration can be treated as
established and reused downstream, which is why LE requires that relied-upon claims trace to a
tool result, file, or source.

## 6. How current coding tools implement loops

This section is the most time-sensitive; product specifics drift quickly and the sources are
largely vendor engineering blogs and documentation rather than peer review.

**Claude and the Claude agent tooling.** The documented loop is gather context, take an
action, verify the result, repeat [18]. Product guidance categorizes loops by their stop
condition, including turn-based loops that stop when the model judges the work done,
goal-based loops that stop when a completion condition is met or a turn cap is reached with a
separate evaluator checking completion, and time or schedule-triggered loops [19]. The
recommended practices map directly onto LE: use a separate reviewer with fresh context because
it is less biased, encode verification steps as reusable skill files, use scripts for
deterministic work to save reasoning budget, and pilot a loop on a small case before running
it at scale. Guidance for long-running agents adds an initializer-versus-worker split, where a
first pass sets up the environment, a progress log, and an initial commit, and subsequent
passes make incremental, individually committed progress so the loop can always revert to a
working state [18].

**OpenAI Codex.** The documented loop is the classic query, tool-call, append-output, re-query
cycle that ends when the model returns an assistant message with no tool calls [20]. The write-
up notes that the loop's cost grows faster than linearly because each step re-sends
accumulated context, mitigated by prompt caching and compaction, and that a separate small
model evaluates completion so the writer is not the grader. Always-on repository context lives
in an agents file, while per-task autonomous loops are driven by a goal command.

**Gemini command-line tooling.** Implements a ReAct-style loop with an explicit session-turn
cap intended to prevent infinite loops, loop detection enabled by default, a next-speaker
check, and context compression triggered at a usage threshold. As with the others, these are
standard guardrails, and as with the others, the specific interface is a moving target; some
of this tooling is being renamed and reorganized, so any exact command should be re-verified.

**Benchmark agents and frameworks.** SWE-agent introduced an agent-computer interface on a
ReAct loop with explicit cost and step caps [14]. General agent frameworks expose cyclic
graphs that require explicit stop conditions on their edges, with a default recursion limit as
the backstop [21]. Earlier autonomous-agent projects were notorious for infinite loops and
goal drift, which is much of why the current generation ships loop detection and hard caps by
default.

## 7. Goal-oriented engineering

The practitioner claim that how a project directs its agents separates success from failure
maps onto concrete, sourced practices. Make success externally verifiable, preferring
deterministic criteria such as tests passed or a score threshold [18]. Decompose the work and
handle one unit at a time, committing after each working increment so there is always a state
to revert to [18]. Ground termination in external verification over self-critique, which is
the design consequence of the self-correction evidence [10,11]. Keep durable memory in a
re-read spec or progress file so the goal survives across many iterations and across context
resets [18]. Separate the maker from the checker using a fresh-context reviewer or a separate
evaluator model [19,20]. These are not competing recipes; they are the same discipline of
giving the loop a reliable signal and a reliable memory.

## 8. Cross-model universality

The three major agent families implement the same fundamental loop and expose analogous
controls: iteration or turn caps, context compaction, loop detection, and evaluator-based
completion checks [18,19,20,21]. The open skill-file format they share means one skill
definition can drive all of them. This is what makes LE portable in principle.

The differences that exist are mostly at the level of product positioning and are reported by
vendors rather than measured in controlled head-to-head studies, so they should be treated as
directional. What is common across models is more decision-relevant than what differs: every
broadly tested model shows context rot [16], so context hygiene is universal; and the
self-correction limitation is a property of current models in general rather than of any one
vendor [10,11]. LE therefore writes its guidance against the shared loop shape and the shared
verifier rule, and confines model-specific details to per-platform installation notes.

## 9. Open questions and what would change these recommendations

LE's recommendations are contingent on the current state of the art, and specific future
results would revise them.

- If trained self-correction in the style of SCoRe [12] generalizes to reliable,
  deployment-time self-correction without external feedback, LE's strict "do not trust
  self-judgment" rule could soften into "trust it within calibrated bounds."
- If context rot is substantially mitigated architecturally [16,17], long single-context loops
  become safer and step caps could rise, reducing the need for aggressive compaction and
  fresh-context restarts.
- The trend in how long a task agents can complete autonomously is itself moving. Work
  measuring the time horizon of tasks agents can finish reliably reports that this horizon has
  been growing on a regular doubling schedule [24]. If that trend holds, the appropriate step
  and turn budgets for autonomous loops should be revised upward over time rather than treated
  as fixed. This is the signal to watch when deciding whether to raise caps.

The right posture is to hold the control structure fixed (verifiable goal, external stop
signal, caps as backstop, checkpoints, decompose on stall) while treating the specific numbers
as parameters to recalibrate as models and tools change.

## References

[1] J. Wei, X. Wang, D. Schuurmans, M. Bosma, B. Ichter, F. Xia, E. Chi, Q. Le, and D. Zhou, "Chain-of-thought prompting elicits reasoning in large language models," arXiv preprint arXiv:2201.11903, 2022 (NeurIPS 2022).

[2] X. Wang, J. Wei, D. Schuurmans, Q. Le, E. Chi, S. Narang, A. Chowdhery, and D. Zhou, "Self-consistency improves chain of thought reasoning in language models," arXiv preprint arXiv:2203.11171, 2022 (ICLR 2023).

[3] S. Yao, J. Zhao, D. Yu, N. Du, I. Shafran, K. Narasimhan, and Y. Cao, "ReAct: Synergizing reasoning and acting in language models," arXiv preprint arXiv:2210.03629, 2022 (ICLR 2023).

[4] A. Madaan, N. Tandon, P. Gupta, S. Hallinan, L. Gao, S. Wiegreffe, U. Alon, N. Dziri, S. Prabhumoye, Y. Yang, S. Gupta, B. P. Majumder, K. Hermann, S. Welleck, A. Yazdanbakhsh, and P. Clark, "Self-Refine: Iterative refinement with self-feedback," arXiv preprint arXiv:2303.17651, 2023 (NeurIPS 2023).

[5] N. Shinn, F. Cassano, A. Gopinath, K. Narasimhan, and S. Yao, "Reflexion: Language agents with verbal reinforcement learning," arXiv preprint arXiv:2303.11366, 2023 (NeurIPS 2023).

[6] S. Yao, D. Yu, J. Zhao, I. Shafran, T. L. Griffiths, Y. Cao, and K. Narasimhan, "Tree of Thoughts: Deliberate problem solving with large language models," arXiv preprint arXiv:2305.10601, 2023 (NeurIPS 2023).

[7] M. Besta, N. Blach, A. Kubicek, R. Gerstenberger, M. Podstawski, L. Gianinazzi, J. Gajda, T. Lehmann, H. Niewiadomski, P. Nyczyk, and T. Hoefler, "Graph of Thoughts: Solving elaborate problems with large language models," arXiv preprint arXiv:2308.09687, 2023 (AAAI 2024).

[8] L. Wang, W. Xu, Y. Lan, Z. Hu, Y. Lan, R. K.-W. Lee, and E.-P. Lim, "Plan-and-Solve prompting: Improving zero-shot chain-of-thought reasoning by large language models," arXiv preprint arXiv:2305.04091, 2023 (ACL 2023).

[9] A. Zhou, K. Yan, M. Shlapentokh-Rothman, H. Wang, and Y.-X. Wang, "Language Agent Tree Search unifies reasoning, acting, and planning in language models," arXiv preprint arXiv:2310.04406, 2023 (ICML 2024).

[10] J. Huang, X. Chen, S. Mishra, H. S. Zheng, A. W. Yu, X. Song, and D. Zhou, "Large language models cannot self-correct reasoning yet," arXiv preprint arXiv:2310.01798, 2023 (ICLR 2024).

[11] R. Kamoi, Y. Zhang, N. Zhang, J. Han, and R. Zhang, "When can LLMs actually correct their own mistakes? A critical survey of self-correction of LLMs," Transactions of the Association for Computational Linguistics, vol. 12, pp. 1417-1440, 2024 (arXiv:2406.01297).

[12] A. Kumar, V. Zhuang, R. Agarwal, Y. Su, J. D. Co-Reyes, A. Singh, K. Baumli, S. Iqbal, C. Bishop, R. Roelofs, L. M. Zhang, K. McKinney, D. Shrivastava, C. Paduraru, G. Tucker, D. Precup, F. Behbahani, and A. Faust, "Training language models to self-correct via reinforcement learning," arXiv preprint arXiv:2409.12917, 2024 (ICLR 2025).

[13] G. Tyen, H. Mansoor, V. Carbune, P. Chen, and T. Mak, "LLMs cannot find reasoning errors, but can correct them given the error location," Findings of the Association for Computational Linguistics: ACL 2024, 2024.

[14] J. Yang, C. E. Jimenez, A. Wettig, K. Lieret, S. Yao, K. Narasimhan, and O. Press, "SWE-agent: Agent-computer interfaces enable automated software engineering," arXiv preprint arXiv:2405.15793, 2024 (NeurIPS 2024).

[15] V. Krakovna, J. Uesato, V. Mikulik, M. Rahtz, T. Everitt, R. Kumar, Z. Kenton, J. Leike, and S. Legg, "Specification gaming: The flip side of AI ingenuity," DeepMind Safety Research Blog, 2020.

[16] K. Hong, A. Troynikov, and J. Huber, "Context Rot: How increasing input tokens impacts LLM performance," Chroma Research Technical Report, July 2025.

[17] N. F. Liu, K. Lin, J. Hewitt, A. Paranjape, M. Bevilacqua, F. Petroni, and P. Liang, "Lost in the middle: How language models use long contexts," Transactions of the Association for Computational Linguistics, 2024 (arXiv:2307.03172, 2023).

[18] Anthropic, "Effective harnesses for long-running agents," Anthropic Engineering, 2025.

[19] Anthropic, "Getting started with loops," Claude blog, June 30, 2026.

[20] OpenAI, "Unrolling the Codex agent loop," OpenAI, 2025.

[21] LangChain, "GRAPH_RECURSION_LIMIT," LangGraph documentation, 2025.

[22] A. Osmani, "Loop Engineering: The Guide for AI Agents," 2026. Consolidating practitioner writing; the framing was popularized in 2026 remarks by B. Cherny (Anthropic) and in posts by A. Karpathy and P. Steinberger. These are non-peer-reviewed practitioner sources.

[23] F. P. B. Osinga, "Science, Strategy and War: The Strategic Theory of John Boyd," Routledge, 2007.

[24] T. Kwa, B. West, J. Becker, et al. (METR), "Measuring AI ability to complete long tasks," 2025.

[25] op7ic, "Adaptive Depth Reasoning Skill," GitHub repository, 2026. Sibling skill and structural template. [Online]. Available: https://github.com/op7ic/Adaptive_Depth_Reasoning_Skill
