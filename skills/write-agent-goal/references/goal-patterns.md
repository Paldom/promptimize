# Goal Patterns — Writing Goal Prompts for Autonomous Agent Runs

Reference for authoring the goal text and stop condition of autonomous runs
(Claude Code `/goal`, Stop-hook loops, and equivalent goal-gated harnesses).
Official-doc facts are cited inline; volatile platform mechanics are version-gated.

## Contents

1. [Goal-contract anatomy](#1-goal-contract-anatomy)
2. [Success-condition design](#2-success-condition-design)
3. [The three exits](#3-the-three-exits)
4. [Evidence discipline](#4-evidence-discipline)
5. [Scope clauses](#5-scope-clauses)
6. [Platform mechanics (verified)](#6-platform-mechanics-verified)
7. [Layering by change frequency](#7-layering-by-change-frequency)

## 1. Goal-contract anatomy

Write the goal as a contract, never a recipe.

- Include exactly five parts: objective, why it matters, hard constraints,
  available tools, and a machine-checkable definition of done
  (https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5).
- Never write a numbered procedure. Prescriptive step lists cause checklist
  blindness on frontier models — the model works the list instead of the goal.
  Reserve explicit steps for the rare case where the exact method matters.
- Describe the destination, not the steps: "resolve the issue end to end" plus
  explicit completion criteria beats a step script
  (https://developers.openai.com/api/docs/guides/prompt-guidance-gpt-5p6).
- For reasoning models, fill six slots: goal, constraints, context,
  audience/output shape, reasoning budget (set as an API parameter, not prose),
  and evaluation criteria.
- Phrase imperatively, not suggestively. State the why and the audience.
- Grant explicit permission to abstain: "say 'cannot determine' rather than
  guess."

## 2. Success-condition design

A condition is not a prompt. A prompt describes work; a condition describes a
verifiable end state that resolves to a clean yes/no. If it cannot, the loop
has no exit.

- Name the check command, not the desired state: "`npm test` exits 0",
  "`git status` is clean". The `/goal` evaluator judges the transcript only and
  cannot run commands — the condition must force check output into the
  transcript. "Tests should pass" lets the worker's word stand
  (https://code.claude.com/docs/en/goal).
- Three-part condition checklist:
  1. The measurable end-state predicate(s) — every check that must hold
     (test result, build exit code, file count, empty queue). One primary
     state plus its constraint checks is the common shape.
  2. The stated check command whose output proves each predicate.
  3. Do-not-touch constraints ("no snapshot files modified").
- Make conditions falsifiable and close the gaming shortcut by protecting
  the **oracle**: when tests verify the work, "make tests pass" invites test
  edits — "make tests pass without modifying files under `tests/`" does not.
  When tests ARE the deliverable (a test migration), protect a different
  oracle instead: test count, coverage threshold, CI green on unchanged
  assertions. Never adjectives ("high quality")
  (https://developers.openai.com/api/docs/guides/prompt-guidance-gpt-5p6).
- Check quality matters: derive checks from repo configuration where
  possible; capture a baseline first (pre-existing modifications, staged and
  untracked files); prefer authoritative checks (CI, lint/AST) over greps
  that miss aliases or match comments.
- Verifiable beats vague: "Lighthouse score >= 90, max 5 attempts" beats
  "optimize the homepage". "A loop without a real check is `while true` with a
  token bill" (https://claude.com/blog/getting-started-with-loops).
- Completion is an objective predicate on state, never the absence of further
  tool calls — "done" can be hallucinated. Require a state-change check.
- Phrasing pattern: imperative + artifact-anchored — "don't stop until the
  digest is posted and STATE.md is updated."

## 3. The three exits

Every goal defines three exits, in priority order. Most agent failures are
stopping-problem failures.

- **Success** — the checkable condition from section 2 holds.
- **Failure ceilings** — the same failure three consecutive times means
  escalate to a human; attempt 4 starts "fixing" things that were never
  broken (https://kenhuangus.substack.com/p/claude-fable-5-part-2-loop-engineering
  — community heuristic, tune per task). Pair it with a **total-attempt
  cap**: alternating failures never trip a same-failure counter but still
  burn the run.
- **Budget ceiling** — a turns/tokens/dollar cap is hit: checkpoint and park
  with partial work preserved, never die mid-edit.
- **Irreversible work needs safe interruption points**: for rotations,
  migrations, deploys — never start a non-interruptible phase without enough
  remaining budget to finish it, and write the rollback/roll-forward rule
  into the goal so a budget exit cannot strand half-completed state.
- Emit sentinel tokens on exit: `TASK_COMPLETE` / `TASK_FAILED`.
- Always write a cap into the condition text itself — "...or stop after 5
  attempts" / "or stop after 25 turns". A goal has no built-in token budget.
- When you control the harness, layer the budgets: global turn cap AND
  per-tool call cap AND token/cost cap AND wall-clock timeout AND cycle
  detection.
- Define any goal loop by five parts: trigger, verifiable goal, tools,
  state/memory, stop rules — with exits success / surrender (budget) /
  hand-off to human.
- Split maker and checker on the stop decision: a separate fresh-context model
  judges "done" (the `/goal` mechanic), never the worker grading itself.

## 4. Evidence discipline

- Demand evidence, not claims — but **bounded and redacted**: the command,
  its exit status, and the relevant non-sensitive lines or an artifact hash.
  Never full log dumps; never a check command that prints secrets
  (https://developers.openai.com/api/docs/guides/prompt-guidance-gpt-5p6).
- Know the evidence limit: a transcript-only evaluator cannot authenticate
  pasted output — a worker can quote a success it never produced. Prefer
  evidence tied to actual tool-result records; where authenticity matters,
  the harness must execute the check itself (that is loop engineering, not
  goal text).
- Include this audit instruction verbatim in the goal text:

  > Before reporting progress, audit each claim against a tool result from
  > this session. Only report work you can point to evidence for... if tests
  > fail, say so with the output. If a step was skipped, state that.

- On a blocked done-condition, instruct: "stop and report the blocker instead
  of working around it."
- Gate on external verification, never self-assessment: tests pass, links
  resolve, CI green, schema validates, diff scoped to the request.
- Run deterministic checks before any LLM judgment; use a judge only for
  interpretation-dependent criteria, and show it only the question plus the
  candidate answer.
- Reflection works only when grounded in a verifier the generator did not
  produce (tests, compiler/linter, schema, retrieved evidence, or a clean-
  context critic). Same-model self-critique rubber-stamps its own error.

## 5. Scope clauses

Scope clauses are the budget. Put them in the goal text itself.

- Name off-limits paths explicitly: "Do not modify files outside
  `/src/pages/home`."
- Prefer several small sequential goals over one giant goal: three goals with
  a crisp check each terminate naturally and give checkpoints between them.
- Why small goals win: per-step reliability compounds — 0.95^10 ~= 60%,
  0.95^20 ~= 36% (arithmetic; https://utkarshkanwat.com/writing/betting-against-agents
  — not provider-official). Shorten chains and verify per step.

## 6. Platform mechanics (verified)

- `/goal` is built into Claude Code v2.1.139+: a wrapper around a
  session-scoped, prompt-based Stop hook. The condition is evaluated each turn
  by a small fast model (default Haiku) that sees only the transcript.
  Subcommands: `clear` / `stop` / `off` / `reset`.
  (as of July 2026 — verify: https://code.claude.com/docs/en/goal)
- Stop hooks block completion via exit code 2 or JSON `{"decision": "block"}`;
  exit 0 allows the stop; stderr from exit 2 is fed back to Claude.
  (as of July 2026 — verify: https://code.claude.com/docs/en/hooks)
- The ~4,000-character condition-length figure is community-reported, not
  confirmed in official docs.
  (as of July 2026 — verify: https://code.claude.com/docs/en/goal)
- Answer four pre-flight questions before starting any goal:
  1. What evidence shows completion?
  2. Who validates it?
  3. After how many consecutive failures do we stop?
  4. When must a human be called (permissions, cost, context)?

## 7. Layering by change frequency

Put each instruction in the layer that matches how often it changes.

- **System layer** (never changes): identity, boundaries, negative
  constraints.
- **Task layer** (changes per goal): the goal, success criteria, and the
  evidence standard.
- **Per-iteration layer** (changes every cycle): what changed, the next
  concrete step, the report format.
- In plan-and-execute loops, the executor prompt carries ONE step plus its
  tools; the goal lives in the plan artifact, not in conversation history.
