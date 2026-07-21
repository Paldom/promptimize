# Reliability: Engineering Agentic Loops

Reference for building agent loops that terminate, verify, and fail safely.
Every control below lives in the harness — code, hooks, CI — never only in the prompt.

## Contents

- [1. Caps live in code, never prompts](#1-caps-live-in-code-never-prompts)
- [2. The effective-bound principle](#2-the-effective-bound-principle)
- [3. Budget mechanics](#3-budget-mechanics)
- [4. No-progress detection](#4-no-progress-detection)
- [5. Deterministic hooks and side-effect safety](#5-deterministic-hooks-and-side-effect-safety)
- [6. Verification](#6-verification)
- [7. Compounding math, done honestly](#7-compounding-math-done-honestly)
- [8. Incidents become regression tests](#8-incidents-become-regression-tests)

## 1. Caps live in code, never prompts

**Never let the model own termination.** ~38% of studied infinite loops had
model-controlled stop conditions (arxiv.org/abs/2607.01641).

- Do not rely on prompt-level caps ("max 3 tries") — they degrade as context
  grows and the instruction gets buried. Enforce limits in the orchestrator.
- Layer independent caps; each catches failures the others miss:
  - Iteration cap: hard `max_iterations` in the harness (~10 is a common default).
  - Wall-clock: `max_execution_time` enforced outside the model.
  - Token/cost budget: cumulative spend tracked across the whole session.
  - Cycle detection: repeated tool+args and state oscillation (section 4).
- Add diff-size and timeout caps to the same orchestrator layer — never the prompt.
- Treat goal completion as an objective predicate over external state, never as
  the absence of tool calls: "done" can be hallucinated; require a state-change check.

## 2. The effective-bound principle

**A bound counts only if it constrains the repeated path itself.**
Source: "When Agents Do Not Stop: Uncovering Infinite Agentic Loops in LLM Agents"
(Hou et al., arxiv.org/abs/2607.01641 — IAL-Scan tool, Agentic Loop Dependence Graph).

- The paper's term is "effective bound". `max_retries=3` inside a function is
  worthless when an outer workflow re-enters that function: the composite loop
  stays unbounded even though every scope looks capped.
- Therefore budgets must be cumulative and session-wide — track attempts across
  the whole loop-dependence graph, not per call site.
- Scale: scanning 6,549 repositories surfaced 68 confirmed infinite agentic
  loops across 47 projects, at 91.9% detection precision.
- Failure-pattern split of the confirmed loops:
  - retry 25%
  - tool-iteration 23.5%
  - multi-agent conversation 20.6%
  - workflow 13.2%
  - reentry 10.3%
  - evaluator 7.4%
- Audit every loop in your system against that list — retries, tool loops,
  agent-to-agent chat, workflow re-entry, evaluator loops — and give each one
  its own effective bound.

## 3. Budget mechanics

**Check pre-flight; fail loud; land soft.**

- Run the budget check PRE-flight, before each LLM call. A post-flight check has
  already burned the tokens it was supposed to protect. Reserve worst-case
  output tokens and in-flight concurrent spend in the check — otherwise a
  "hard" cap is exceeded by calls already running.
- Budget breach = raised exception, never a return code. Return codes get
  ignored; an exception is far harder to swallow silently — but it can still
  be caught, so never route budget exceptions through a generic retry
  handler.
- Soft-stop before hard-stop: when the budget is nearly exhausted, inject
  "budget nearly exhausted — wrap up" so the agent finishes cleanly instead of
  dying mid-edit. The hard cap still follows if it ignores the warning.
- Checkpoint, not cliff: a threshold breach triggers a structured checkpoint
  plus human review, with partial work preserved — never a silent discard of
  everything done so far.
- Durable recovery for external effects: a crash between a side effect and
  its checkpoint is an unknown outcome. Write the intent record before
  acting, reconcile on resume, and fence concurrent resumed supervisors with
  leases. Local state files and git cover local artifacts only — this is the
  territory durable-execution engines exist for.

## 4. No-progress detection

**Detect loops from trace shape — deterministically, no LLM judge needed.**

- Same tool + identical arguments for the 3rd consecutive call → exit with a
  diagnostic (a tunable default, per tool — not a law).
- Make detectors state-aware to avoid false positives: compare normalized
  arguments and results (reordered JSON fields or nonces defeat string
  equality); exempt legitimate polling — changing results are progress; one
  backtrack is not a loop. Require a repeated cycle with no change in a
  defined task-state fingerprint.
- Oscillation between two states is the same detector family — catch A→B→A→B
  flips, not just exact repeats.
- Leading indicator: context/tokens climbing linearly while task state does not
  change. Compact context at ~85–90% utilization to break monotonic-growth
  re-entry.
- 3-strike escalation (default — tune per workload): three failed test/lint
  attempts → lock the loop and escalate to a human. This is the control that keeps a 1-line fix from
  becoming an 18,000-line diff.

## 5. Deterministic hooks and side-effect safety

**Put guard code, not guard prose, around every tool call.**

- Use Claude Code `PreToolUse`/`PostToolUse` hooks (code.claude.com/docs/en/hooks):
  deterministic code runs around each call, and a blocking result (exit code 2 /
  `decision: "block"`) stops execution regardless of what the model decided.
- Keep a per-tool, per-session invocation counter in the hook (loop-guard
  pattern) and block any tool that exceeds its budget.
- Put idempotency keys on every side-effecting tool call — a retried call must
  not double-write or double-charge. The key is only a request: deduplication
  must be enforced receiver-side or in a durable layer to be a guarantee.
- Place guardrails at all three loop boundaries:
  - input: a small fast model pre-screens the request;
  - tool: pre-call block, post-call rewrite;
  - output: checked before the user ever sees it.

## 6. Verification

**Exit gates are external. The builder never grades its own work.**

- Gate loop exit on external verifiers only: tests pass, CI green, schema
  validates, links resolve, diff scoped to the request — never self-assessment
  (RubricMiddleware grader loop with retry-on-feedback:
  langchain.com/blog/the-art-of-loop-engineering).
- Maker–checker split — including on the stop condition itself: a separate,
  fresh-context model judges "done" (the `/goal` mechanic in Claude Code/Codex:
  addyosmani.com/blog/loop-engineering/, code.claude.com/docs/en/goal).
  Fresh context decorrelates the check; it does not make it proof — the
  deterministic gates above remain the gate of record.
- Frame the verifier adversarially: "try to disprove that the criteria are
  met", and audit for letter-vs-spirit gaming — reward hacking is a measured
  behavior, not a hypothetical (metr.org/blog/2026-06-26-gpt-5-6-sol/).
- The verifier checks criteria only. Never prompt it to "verify and improve" —
  fixing collapses the maker–checker split.
- Run deterministic checks before any LLM judge: schema validation, required
  tool order/args, final-state diffs, loop limits, redundant-call rate
  (>2 identical calls), trace-shape anomalies. Reserve judges for
  interpretation-dependent checks, calibrated against human labels.
- Evaluate the trajectory, not just the final answer — layer final-response,
  trajectory, and single-step evals. tau-bench shows correct-looking answers
  sitting on top of wrong database state; pass^k exposes instability that a
  single run hides.

## 7. Compounding math, done honestly

**Per-step reliability decays exponentially with chain length.**

- 95% per-step reliability compounds to ~60% over 10 steps and ~36% over 20:
  0.95^10 ≈ 0.60, 0.95^20 ≈ 0.36. Even 99% per step drops to ~82% over 20.
- This is arithmetic — present it as arithmetic. The canonical write-up is
  Utkarsh Kanwat, "Betting against agents"
  (utkarshkanwat.com/writing/betting-against-agents), NOT a provider doc;
  Anthropic says only the qualitative "compounding errors". Do not attribute
  the numbers to a provider.
- Implications:
  - Shorten chains — every removed step buys back reliability.
  - Verify per step: built-in test feedback is why coding agents beat
    open-ended agents.
  - Prefer a workflow/graph over an open loop whenever the path is known;
    reserve the loop for genuinely unknown paths.

## 8. Incidents become regression tests

**Every production failure is a future CI gate.**

- Pipeline: capture the trace → label → dedupe → add to a versioned golden
  dataset → gate CI on it.
- Freeze broken trajectories and replay them after any prompt or tool change —
  the bug you fixed is the bug most likely to return.
- Practitioner pattern: a ~200-input golden set, rerun weekly, catches drift
  between incidents.
