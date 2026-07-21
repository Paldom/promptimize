---
name: evaluate-prompts
description: Builds standalone prompt evaluations - test sets from real failures, paired A/B comparisons with sample-size math, deterministic checks before LLM judges, judge calibration and bias controls. Use when asked which prompt is better, to A/B test or benchmark prompts, or to set up LLM-as-judge scoring. Not for rewriting prompt text, agent goals, or loop runtime design.
argument-hint: <prompts to compare or eval question>
---

# evaluate-prompts

Answers "is prompt B actually better than prompt A?" with a defensible
protocol instead of vibes. Fixes the standard failure chain: a synthetic
happy-path test set, an uncalibrated 1–10 judge that mean-reverts to 7,
an N too small to detect anything, and a shipped regression.

## When NOT to use

- **Rewriting the prompt** → `optimize-prompt` (it performs a quick check of
  its own rewrite against an existing set; come here when the eval itself is
  the deliverable).
- **Agent-run goal text** → `write-agent-goal`.
- **Agent loop / trajectory harness design** → `engineer-agent-loop`.
- **Code tests, essays, hiring, or product decisions** — evaluation of things
  that aren't prompt variants.
- Model-vs-model shootouts with no prompt variable — this skill owns
  evaluation when prompt text is the controlled variable or the eval
  protocol is the deliverable. Mixed "rewrite it AND prove it better":
  `optimize-prompt` rewrites, then return here for the verdict. Agent
  trajectory-scoring methods live here; wiring them into a harness is
  `engineer-agent-loop`.

## Workflow

1. **Fix the question first.** One primary metric plus non-regressable
   guardrails (latency, cost, refusal rate, safety subset) — written down
   before anything runs.
2. **Build the set from reality — and split it.** 30–100 real production
   cases, ~80% real / ~20% adversarial, never LLM-synthesized for the
   verdict. Hand-grade ~30 outputs and cluster the failures first — the
   harness operationalizes what error analysis found. Then separate roles:
   failure-derived cases drive development and regression; the shipping
   verdict needs a locked, representative holdout that the rewrite (and any
   optimizer) never saw — a failure-only set measures "old bugs fixed", not
   production lift.
3. **Choose scorers deterministic-first.** Exact match, regex, schema
   validation, label match. An LLM judge is the escape hatch for
   interpretation-dependent criteria only.
4. **Pair everything.** Same inputs through both variants, per-case diffs,
   positions swapped for pairwise scoring. The unit of analysis is the case,
   not the generation — repeated samples per case are clustered, not
   independent; record decoding config and randomize run order. Fast
   directional check: 50 blind pairs — 38/50 is signal, 27/50 is noise
   (triage heuristic only).
5. **If a judge is unavoidable, calibrate it.** Per-criterion boolean
   pass/fail with evidence quotes (no Likert); a different — ideally
   stronger — model family, blind to variant identity, version-pinned;
   calibrated against human labels (≥30 sanity, 100–250 proper) reporting
   agreement AND chance-corrected kappa or TPR/TNR — agreement alone
   overstates; ≥80% position-swap consistency (default — tune).
6. **Size N with a power calculation, before running.** Parameterize by
   baseline, minimum detectable effect, α, power, and pairing — paired
   designs need less, driven by discordance; the popular ~200/~800-per-variant
   floors are order-of-magnitude planning figures, not laws. Pre-commit N (no
   peeking). Predeclare the minimum worthwhile effect: the CI must clear it,
   not just zero; guardrail metrics get explicit non-inferiority margins.
7. **Report and institutionalize.** Effect size + CI + per-case diffs; state
   what this N cannot detect; freeze new failures into the regression set;
   hash prompt + model + decoding config + dataset version + scorer/judge
   versions into the variant ID; change one thing at a time.

## Output spec

Either an eval design (set composition, scorers, judge spec + calibration
plan, N and stats plan) or an executed comparison: verdict with effect
size/CI, per-criterion breakdown, stated detection limits, and the regression
cases to freeze. `$ARGUMENTS`, when present, names the prompts or question
under test.

## Gotchas

- Raw judge–human agreement overstates chance-corrected agreement by tens of
  points (kappa deflation) — report kappa, and remember humans agree only
  ~63–66% themselves.
- Consistency is not correctness: a judge can be 0.99 self-consistent and
  severely position-biased at the same time.
- If an automated optimizer produced the winning variant, check for eval
  leakage (did it select on data the baseline never saw?) and prompt bloat
  before believing its delta.
- External benchmarks decay: audits found ~30% of one major coding
  benchmark's tasks broken — audit before trusting deltas on any borrowed
  benchmark.
- Privacy: redact secrets/PII from transcripts before sending them to any
  external judge or eval service — and measure whether redaction changes
  scores.
- Candidate outputs are untrusted input to the judge: delimit them hard,
  give the judge no tools, require structured verdicts, and test it with
  injection canaries.
- A regression set you repeatedly tune against overfits (winner's curse) —
  keep a rotating untouched holdout beside it.

## Pointers

- `references/eval-protocol.md` — the full protocol: set construction,
  scoring discipline, judge pitfalls with primary-source numbers, sample-size
  floors, statistics kit, regression discipline.
