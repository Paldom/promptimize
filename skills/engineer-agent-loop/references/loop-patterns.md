# Loop Patterns for Agentic Systems

Reference for engineering agentic loops: choosing the loop shape, canonical
patterns with verified citations, and the discipline that keeps each shape
working. Reuse figures only with the primary URLs cited inline.

## Contents

- [1. Pattern selection by path-knowability](#1-pattern-selection-by-path-knowability)
- [2. Canonical patterns and citations](#2-canonical-patterns-and-citations)
- [3. Plan-and-execute recipe](#3-plan-and-execute-recipe)
- [4. Reflection discipline](#4-reflection-discipline)
- [5. Multi-agent criteria](#5-multi-agent-criteria)
- [6. Loops vs graphs vs durable execution](#6-loops-vs-graphs-vs-durable-execution)
- [7. The Ralph pattern (prior art)](#7-the-ralph-pattern-prior-art)

---

## 1. Pattern selection by path-knowability

Select the loop by how knowable the solution path is, not by how agentic the
product should feel.

- Path unknown or exploratory → **ReAct**. Let each observation steer the next
  action.
- Path knowable, multi-step, needs audit or approval → **plan-and-execute**.
  The plan is the reviewable artifact.
- Steps fixed for every input → **hardcoded pipeline, no agent**. Do not pay
  loop overhead for a workflow.
- Start at the lowest complexity that works. Upgrade on observed symptoms,
  never preemptively.

Upgrade signals (ReAct → plan-and-execute):

- The agent re-derives the same plan on every request.
- The agent loses the goal mid-run. One analysis (a single Medium source —
  treat as heuristic, not law) puts ReAct coherence loss past ~4 tool calls.

## 2. Canonical patterns and citations

- **ReAct** — Yao et al. 2022, https://arxiv.org/abs/2210.03629. Interleaved
  reason → act → observe. The exploratory default.
- **Reflexion** — Shinn et al., NeurIPS 2023, https://arxiv.org/abs/2303.11366
  (frequently miscited; this is the correct ID). Three roles: the **Actor**
  generates, the **Evaluator** scores, and **Self-Reflection** writes verbal
  lessons into memory for the next attempt. The paper reports 91% pass@1 on
  HumanEval with GPT-4.
- **Plan-and-Solve** — ACL 2023, https://arxiv.org/abs/2305.04091. Explicit
  plan-then-solve prompting.
- **Anthropic, "Building Effective Agents"** —
  https://www.anthropic.com/engineering/building-effective-agents. The
  canonical agent loop: **gather context → take action → verify work →
  repeat**. Workflow taxonomy includes orchestrator-workers,
  evaluator-optimizer, and parallelization.
- **Anthropic multi-agent research system** —
  https://www.anthropic.com/engineering/built-multi-agent-research-system
  (2025-06-13). Orchestrator-worker in production: an Opus 4 lead with
  Sonnet 4 subagents beat single-agent Opus 4 by 90.2% on an internal eval —
  at ~15x the tokens of a chat session. Always quote the cost with the win.
  Guidance: 3–5 parallel subagents, 3+ parallel tool calls each, up to 90%
  research-time reduction.
- **Magentic-One** — https://arxiv.org/abs/2411.04468. Dual-loop orchestrator;
  see §6.

## 3. Plan-and-execute recipe

- Planner emits 5–8 ordered steps, each with an explicit success condition.
- Executor sees ONE step plus that step's tools. The goal lives in the plan
  artifact, not in conversation history.
- Replanner reviews after each step. It is load-bearing: a static plan with no
  replan gate = stale-plan failure, the #1 failure mode of this pattern.
- Replan on policy (preserve the critical path), not on vibes.
- Run a cheap goal-checker every few steps; it catches drift before it
  compounds.
- One step consuming >10 tool calls = stuck or under-specified. Split the
  step.

## 4. Reflection discipline

- Reflect only against a verifier the generator did not produce: unit tests,
  compiler/linter, schema validation, retrieved evidence, or a separate critic
  with clean context.
- Same-model self-critique rubber-stamps its own output (the "FlipFlop
  effect" / "coherence trap"). Do not build a loop on it.
- Verifier hierarchy: deterministic checks first, LLM-judge last. The judge
  sees only the question plus the candidate answer. Precise, testable rubrics
  beat opinions.
- Economics: reflection costs 2–3x tokens — reserve it for checkable or
  high-stakes outputs. Gains land in iterations 1–2. Cap at 3–5 trials. On
  non-convergence, return the best-scoring attempt, not the last one.
- Lesson format: 50–150 tokens (≤3 sentences), citing the tool name and the
  observed error string. No generic advice. Prepend the lesson to the next
  attempt (the Reflexion Self-Reflection role, §2).

## 5. Multi-agent criteria

- Single agent is the default. Escalate only on a concrete, observed failure:
  context-window pressure, a serial-latency wall, or genuine heterogeneity of
  subtasks.
- Use orchestrator-worker only when subtasks cannot be predicted until runtime
  (Anthropic's criterion,
  https://www.anthropic.com/engineering/built-multi-agent-research-system).
  Fixed, predictable subtasks → hardcode the decomposition and parallelize; no
  orchestrator.
- Parallelize independent tool calls; the price is dependency tracking and
  failure handling.
- Scope permissions per worker and keep an audit trail. Permissioning, not
  reasoning, is the measured bottleneck.
- Budget the escalation honestly: the flagship 90.2% improvement (§2) cost
  ~15x the tokens.

## 6. Loops vs graphs vs durable execution

The control-flow debate: how much structure belongs in code versus in the
prompt.

- "Agents need control flow, not more prompts" —
  https://bsuh.bearblog.dev/agents-need-control-flow/ (Hacker News item
  48051562, 590 points, 2026-05). Argues the structure agents lack is
  programmatic control flow, not longer prompts.
- Armin Ronacher, "The Coming Loop" —
  https://lucumr.pocoo.org/2026/6/23/the-coming-loop/. Separates the agent
  loop from the harness loop.
- Long runs (hours, 200+ steps) need durable, checkpointed state with
  resumability and compaction — the loop must survive a context reset.
  Ronacher's "Absurd In Production" (lucumr.pocoo.org, 2026-04-04) covers
  durable checkpointed loops in production.
- Dual-loop for long horizons: an outer strategic planner monitors while an
  inner ReAct loop executes. On inner stall, the outer loop resets strategy
  instead of blindly retrying — the Magentic-One shape
  (https://arxiv.org/abs/2411.04468).
- Whatever the shape, keep a deterministic scaffold around the model:
  plan/checkpoint before any write, diff and revert after tool calls, isolated
  worktrees or sandboxes, and an immutable event ledger.

## 7. The Ralph pattern (prior art)

The maximal "control flow in code" position: the entire outer loop is bash.

- Source: https://ghuntley.com/ralph/ (2025-07-14). "Ralph is a technique...
  a Bash loop":

  ```bash
  while :; do cat PROMPT.md | claude-code ; done
  ```

- Ralph's rules: one task per loop; allocate specs/plans deterministically;
  search the codebase before assuming something is unimplemented; NO
  placeholder implementations; one agent for build/test, parallel subagents
  for other work; capture test intent; accept eventual consistency.
- Counterweight: Ronacher's "Agent Psychosis" (lucumr.pocoo.org, 2026-01-18)
  critiques Ralph-style slop loops. Read both before adopting the pattern.
