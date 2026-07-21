# Prompt Evaluation Protocol

Verification protocol for prompt changes. Run this before believing — or shipping —
any "improved" prompt. Numbers carry inline primary-source citations.

## Contents

1. [Minimal before/after protocol](#1-minimal-beforeafter-protocol)
2. [Deterministic checks first](#2-deterministic-checks-first)
3. [Scoring discipline](#3-scoring-discipline)
4. [LLM-judge pitfalls](#4-llm-judge-pitfalls)
5. [Statistics when it matters](#5-statistics-when-it-matters)
6. [Regression discipline](#6-regression-discipline)

---

## Minimum bar (quick checklist)

- [ ] Eval set built from 30–100 real failures, not synthetic
- [ ] Deterministic scorers wherever possible; judge only where unavoidable
- [ ] Paired runs on identical inputs; positions swapped for pairwise
- [ ] Boolean per-criterion scoring with evidence quotes, no Likert
- [ ] Judge calibrated against human labels (≥80% agreement, kappa reported)
- [ ] N pre-committed; bootstrap CI on the diff excludes zero
- [ ] Failures frozen into the regression set; variant ID hashes prompt+model+config

## 1. Minimal before/after protocol

- Build the eval set from 30–100 real production failures/traces. Never
  LLM-synthesize cases for the verdict — synthetic sets skew happy-path
  (clearly labeled synthetic challenge cases are fine as a separate slice).
- Mix ~80% real cases, ~20% adversarial.
- Split roles to avoid leakage: development/error-analysis cases (from
  failures; the rewrite may see them) vs a locked, representative holdout
  (untouched by prompt development or optimizer selection — the only basis
  for a shipping verdict) vs the frozen regression suite. A failure-derived
  set alone estimates "old bugs fixed", not production lift.
- Before touching any harness, hand-grade ~30 real outputs and cluster failures
  by type (Hamel Husain's error-analysis-first method). Frameworks operationalize
  what manual analysis found; they don't replace it.
- Compare paired, always: same inputs, same scorer, both variants, with per-case
  diffs. Never Monday-baseline vs Tuesday-candidate.
- Fast directional check: 50 blind pairwise samples. Rewrite wins 38/50 (76%) =
  strong signal; 27/50 = noise — stop and rethink. A coin-flip triage
  heuristic only; decisions use the power-calculated N below.
- Prefer downstream behavioral metrics when they exist: reply rate, resolution
  rate, user-edit rate, code-accepted-unmodified.
- Fix one primary metric plus non-regressable guardrails (latency, cost, refusal
  rate, safety subset) before running anything.

## 2. Deterministic checks first

- Score with deterministic checks wherever the criterion allows: exact match,
  regex, JSON schema validation, label match.
- "LLM-as-judge is an escape hatch, not a default." Reach for a judge only when
  no deterministic check can express the criterion.
- For agent prompts, score trajectories — tool choice, arguments, ordering,
  fabricated execution — not just final answers.

## 3. Scoring discipline

- Kill 1–10 / Likert scales: judges mean-revert to 7–8.
- Use per-criterion boolean pass/fail for rubric judgments, each verdict
  justified with an evidence quote from the output. Decomposed atomic yes/no
  questions correlate better with human judgment — the strongest-consensus
  finding across sources. (Inherently comparative or continuous criteria
  still warrant pairwise, ordinal, or direct task metrics.)
- Rubric quality beats judge model: "weak judge + great rubric beats great judge
  + weak rubric."
- Rubric mechanics: per-level anchors, one prompt per criterion, 3–5 dimensions
  max. Tightening the spec cut judge variance 30–50% (single-source figure).

## 4. LLM-judge pitfalls

- Raw judge–human agreement overstates chance-corrected Cohen's kappa by
  33.8–41.3pp on MT-Bench (~541K judgments, 21 judges across 9 providers) —
  "Reliability without Validity" ([arXiv 2606.19544](https://arxiv.org/pdf/2606.19544)).
  Report TPR/TNR + kappa, never raw agreement.
- Consistency is not correctness (the paper's "consistency–bias paradox"):
  Qwen3-8B shows 0.992 test-retest self-consistency yet the study's worst
  position bias (0.192) — same paper.
- Frontier judges (Gemini-2.5-Pro, GPT-5) fail to maintain consistent
  preferences in ~25% of difficult cases — "Are We on the Right Way to
  Assessing LLM-as-a-Judge?" (Sage suite,
  [arXiv 2512.16041](https://arxiv.org/pdf/2512.16041)). Self-generated rubrics
  cut inconsistency by 16.1% / 11.0%.
- Human raters themselves agree only 63% (MT-Bench) / 66% (AlpacaFarm) vs 50%
  random — Zheng et al. 2023, as cited by arXiv 2512.16041. Do not demand more
  from a judge than humans achieve; ~85% judge–human agreement is an upper
  bound under favorable conditions, not a guarantee.
- Position bias: always run both orderings (swap A/B positions); require ≥80%
  swap-consistency, otherwise move to a stronger judge. The bias is driven by
  quality gap, not response length
  ([IJCNLP 2025](https://aclanthology.org/2025.ijcnlp-long.18/)).
- Calibrate the judge against 100–250 human-labeled examples (minimum 30 for a
  sanity check); gate at ≥80% agreement AND report the chance-corrected
  kappa / TPR-TNR alongside it — the agreement gate is a screen, never the
  reliability claim (see kappa deflation above). Thresholds are defaults to
  tune.
- Judge hygiene: different — ideally stronger — model family than the
  generator; blind to variant identity; pin the judge version; re-calibrate on
  any change; spot-check 5–10% of production scores.
- Judge cost control: cascade — a cheap judge scores the bulk, a frontier judge
  re-scores only the disagreements.
- Benchmarks fail too: OpenAI's audit found ~30% of SWE-Bench Pro's 731 public
  tasks broken
  ([openai.com](https://openai.com/index/separating-signal-from-noise-coding-evaluations/)),
  and the same model can shift up to 14 rank positions between benchmarks.
  Audit any external benchmark before trusting deltas on it.

## 5. Statistics when it matters

- Sample-size planning (binary metric, 60% baseline): ~200/variant for a
  10pp lift, ~800 for 5pp, ~5,000 for 2pp — order-of-magnitude figures from
  a practitioner guide (masterprompting.net); assumptions (sidedness,
  pairing, power) move them severalfold — two independent arms at 80% power
  need roughly 350+/arm for 10pp. Run a parameterized power calculation;
  paired designs need less, driven by discordance. 50 binary cases resolve
  only 10–15pp swings.
- No peeking: checking p<0.05 every 10 samples and stopping early inflates the
  false-positive rate to ~30% (reported). Pre-commit N before the run.
- Unit of analysis is the case: repeated generations per case are clustered,
  not independent samples; record decoding configuration; randomize run
  order.
- Bootstrap a 95% CI on the score difference (500–1,000 resamples). CI
  containing zero = don't ship.
- Test choice: Wilcoxon signed-rank (paired, non-normal scores), McNemar
  (paired binary), paired t-test (latency/cost).
- Effect-size bands (continuous scores only — not binary metrics; rules of
  thumb, not decision laws): d < 0.2 ignore; 0.5–0.8 meaningful; ≥ 0.8
  large. Ship decisions rest on the predeclared minimum worthwhile effect
  plus guardrail non-inferiority margins, not on d alone.

## 6. Regression discipline

- Fold every new production failure into a frozen regression set; refresh
  roughly monthly.
- Version everything: hash prompt + model + decoding config + dataset
  version + scorer/judge versions into the variant ID; keep frozen replay
  sets so old runs stay reproducible.
- A regression set you repeatedly tune against will be overfit (winner's
  curse): keep a rotating, untouched holdout beside the frozen suite for
  verdicts.
- Change one thing at a time — a variant that bundles prompt, model, and config
  changes cannot tell you which change moved the metric.
- Pre-write rollout and rollback criteria; wire evals in as CI gates.
- Ship pipeline: offline gate → judge calibration → paired A/B → canary/shadow
  → production monitoring.
