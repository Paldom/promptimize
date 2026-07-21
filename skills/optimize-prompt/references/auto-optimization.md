# Automated Prompt Optimization

When and how to escalate from hand-tuning to automated prompt optimization.

## Contents

1. [Prerequisite gate](#1-prerequisite-gate)
2. [Framework-free meta-prompting recipe](#2-framework-free-meta-prompting-recipe)
3. [Escalation ladder: BootstrapFewShot → MIPROv2 → GEPA](#3-escalation-ladder-bootstrapfewshot--miprov2--gepa)
4. [Verified headline numbers](#4-verified-headline-numbers)
5. [Pitfalls](#5-pitfalls)
6. [When hand-tuning still wins](#6-when-hand-tuning-still-wins)

## 1. Prerequisite gate

**No metric + no held-out data = do not automate.** Hand-tune and build the eval
first. Building the metric is the hard part — the optimizer only does wording.

- Separate spec from implementation: the human declares inputs/outputs (a
  signature) and writes the metric; the optimizer searches over phrasing.
- Typical data need: **20–100 labeled examples** for GEPA-class optimizers;
  ~50–100 representative cases plus a stable judge, or the optimizer overfits
  to noise. (The GEPA repo self-claims it works from 3 examples — treat as an
  unverified self-claim: https://github.com/gepa-ai/gepa)
- If the metric is an LLM judge, stabilize it before optimizing (see judge
  hygiene in [Pitfalls](#5-pitfalls)).
- Eval harness option: promptfoo — MIT, still open source and active after
  OpenAI's acquisition announced 2026-03-09
  (https://openai.com/index/openai-to-acquire-promptfoo/)
  (as of July 2026 — verify: https://www.promptfoo.dev/).

## 2. Framework-free meta-prompting recipe

Cheapest automation: no framework, one strong model in a loop. Best when the
failure is **conceptual** — a missing instruction, conflated tasks, an
underspecified format.

- Hand a strong teacher model: (1) the current prompt, (2) failing examples,
  (3) prior attempts that scored **worse**. Ask it to **diagnose, then
  rewrite** — one reflective rewrite per round.
  (https://futureagi.com/blog/automatic-prompt-optimization/)
- Loop mechanics (Evidently): the LLM improves the prompt using the previous
  iteration's errors; split data **40/40/20** train/val/test; early stopping;
  **multiple restarts**.
  (https://www.evidentlyai.com/blog/automated-prompt-optimization)
- **Keep attempt memory.** Stateless iterative rewriting causes catastrophic
  forgetting / semantic drift — tone and format rules erode fastest. Feeding
  back prior worse attempts is what prevents re-walking dead ends.

## 3. Escalation ladder: BootstrapFewShot → MIPROv2 → GEPA

DSPy frames this as "programming — rather than prompting — language models":
declare a signature + metric, then compile
(https://github.com/stanfordnlp/dspy). Run the cheapest rung first; step up
only when the current rung plateaus on the held-out set.

1. **BootstrapFewShot** — cheap; bootstraps few-shot demos from the program's
   own successful traces. Start here.
2. **MIPROv2** — Bayesian joint search over instructions + demos. Step up when
   demos alone stop helping. Example API:
   `dspy.MIPROv2(metric=..., auto="medium").compile(program, trainset, valset)`
   (as of July 2026 — verify: https://dspy.ai/diving-deeper/choosing-an-optimizer/).
3. **GEPA** — reflective mutation with Pareto-frontier selection
   (https://github.com/gepa-ai/gepa, https://arxiv.org/abs/2507.19457). Step up
   only when ALL hold: MIPROv2 has plateaued; the metric can return **textual
   feedback** (not just a scalar); the pipeline is complex. Foot-gun: the
   **reflection model must be strong** — a small reflection LM produces
   useless critiques.

Picking by problem shape (run cheapest first):

- Single prompt + scalar metric → textual-gradient methods (ProTeGi, OPRO).
- Nameable conceptual gap → one meta-prompt rewrite (section 2).
- Pipeline / multi-objective → GEPA.

Also:

- **Cross-model:** recompile per target model instead of rewriting by hand;
  teacher-student works (a big model optimizes prompts for 3B–8B local models,
  which are hyper-sensitive to wording).
- GEPA's `optimize_anything` evolves whole harnesses; the biggest agent gains
  reportedly come from structural changes (planner/executor split, tool-output
  filtering), not prompt tweaks (as of July 2026 — verify:
  https://github.com/gepa-ai/gepa). Integrations exist in MLflow
  (`mlflow.genai.optimize_prompts()`), Opik, and ADK (`adk optimize`)
  (as of July 2026 — verify: https://github.com/gepa-ai/gepa).

## 4. Verified headline numbers

Only figures verified against primary sources; everything else from vendor
restatements, single practitioners, or single X posts is excluded.

- **GEPA vs GRPO (RL):** "outperforms GRPO by 6% on average and by up to 20%,
  while using up to 35x fewer rollouts" — abstract verbatim,
  https://arxiv.org/abs/2507.19457.
- **GEPA vs MIPROv2:** "by over 10% (+12% on AIME-2025)" — same paper, verbatim.
- **CAPO:** 11/15 wins, up to +21% — peer-reviewed,
  https://proceedings.mlr.press/v293/zehle25a.
- **AutoPDL:** +9.06 accuracy — peer-reviewed,
  https://proceedings.mlr.press/v293/spiess25a.

## 5. Pitfalls

- **Eval leakage in "automation wins" comparisons:** many let the optimizer
  see labeled splits the manual baseline never saw
  (https://arxiv.org/abs/2603.25169). Compare on data neither side touched.
- **Prompt bloat** is the top overfitting failure mode — add length penalties.
- **Overfitting to small val sets:** keep a held-out test split; never select
  the winner on a small random val subset.
- **Never manually edit an optimized prompt afterward** — re-run the optimizer
  instead.
- **Judge hygiene:** pin judge model versions and seeds; use binary or 1–5
  Likert with CoT, not 1–100; temp=0 is not deterministic. Raw agreement
  overstates judge reliability — kappa deflation of 33.8–41.3pp on MT-Bench
  (https://arxiv.org/abs/2606.19544) — and frontier judges flip preferences in
  ~25% of difficult cases (https://arxiv.org/abs/2512.16041).
- **Security re-audit before deploy:** the same optimizers double as
  red-teamers; an optimized prompt is a new attack surface.

## 6. When hand-tuning still wins

- Exploration phase, before a metric exists.
- Subjective or creative output that resists automatic scoring.
- Error-detection sub-metrics (the judge itself is the weak link).
- One-off tasks — optimizer setup costs more than it returns.
