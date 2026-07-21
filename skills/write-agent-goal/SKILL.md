---
name: write-agent-goal
description: Drafts or fixes the goal text for an autonomous agent run (/goal, unattended sessions) - checkable success conditions, evidence requirements, scope and permission limits, retry and budget exits. Use when writing an agent goal, run definition of done, or run success criteria, or when a goal gets gamed or marked done early. Not for generic project goals, prompt rewrites, or loop runtime design.
argument-hint: <task to turn into an agent goal>
---

# write-agent-goal

Turns a task into a goal contract an autonomous agent can be held to: a
verifiable end state, the evidence that proves it, the boundaries the run must
respect, and the exits that stop it. Fixes the two failures goal-gated runs
actually die of: a run "marked done" on the worker's own word, and a run that
can never satisfy its condition because the condition was never checkable.

## When NOT to use

- **Runtime symptoms** — the loop stalls, repeats itself, drifts, or ignores
  caps → `engineer-agent-loop`. Diagnose the harness first; the goal text may
  be fine.
- **Ordinary prompt work** — rewriting a system/task prompt → `optimize-prompt`.
- **Comparing or benchmarking prompts** → `evaluate-prompts`.
- **Human/project goal-setting** — OKRs, tickets, roadmap "definition of done"
  with no autonomous run attached.
- **Compound asks** ("rewrite the goal AND add harness caps"): write the goal
  text here, then hand off to `engineer-agent-loop` for enforcement.

## Workflow

1. **Extract the contract.** Five parts: objective, why it matters, hard
   constraints, available tools, machine-checkable definition of done. Not a
   numbered procedure — step lists cause checklist blindness on frontier
   models; describe the destination (exception: when the exact method is
   itself a requirement, the method becomes a constraint).
2. **Design the success condition.** Apply the checklist from
   `references/goal-patterns.md`: the measurable end-state predicates (often
   one primary state plus constraint checks); the named check command whose
   output proves each; do-not-touch constraints. Close gaming shortcuts by
   protecting the *oracle*: when tests verify the work, forbid weakening them
   ("tests pass **without modifying files under tests/**"); when tests ARE
   the work (a test migration), protect a different oracle instead (test
   count, coverage, CI on unchanged assertions). Ban adjectives ("high
   quality", "improved"). If the task is genuinely subjective — no machine
   check exists — say so and gate on explicit human approval or a declared
   measurable proxy; never fake verifiability.
3. **Set the exits, in priority order.** Success (the condition holds);
   failure ceilings — same failure ~3 consecutive times AND a total-attempt
   cap (alternating failures burn budget without tripping a same-failure
   counter) → stop and escalate with a diagnosis; budget ceiling
   (turns/tokens/cost → checkpoint, park partial work, report). Write the
   caps into the condition text — a goal has no built-in budget. For
   irreversible or multi-stage work (rotations, migrations, deploys): never
   start a non-interruptible phase without budget to finish it, and name the
   rollback/roll-forward rule so a budget exit cannot strand half-done state.
4. **Bound the blast radius.** Explicit off-limits paths; side-effect limits
   (no deploys, no spending, no git push, no external posts — whatever
   applies); an approval gate for irreversible or outward-facing actions;
   secrets never quoted into the goal or printed by its check commands. Say
   the quiet part in the output: these clauses reduce *accidental*
   violations only — "must never touch prod", hard cost limits, and trusted
   completion need sandboxing, credential separation, and supervisor caps
   (`engineer-agent-loop`).
5. **Set the evidence standard.** Require bounded, redacted evidence: the
   command, its exit status, and the relevant non-sensitive lines or an
   artifact hash — never full log dumps, never secret-printing commands.
   Include the audit instruction (verbatim in
   `references/goal-patterns.md`); on a blocked condition, instruct the
   agent to stop and report the blocker rather than work around it. Note the
   limit honestly: a transcript evaluator cannot authenticate pasted output —
   where authenticity matters, the harness must run the check itself.
6. **Size it.** Prefer several small sequential goals with a crisp check each
   over one giant goal — reliability compounds per step. Keep the condition
   compact (community-reported ~4,000-char cap for `/goal`).
7. **Read it back as the laziest compliant agent.** Can the letter be
   satisfied while the spirit fails (edit the tests, weaken the assertion,
   declare victory without running the check)? Patch every shortcut found,
   then answer the four pre-flight questions in the reference.

## Output spec

A paste-ready goal text containing: objective + why; hard constraints
including off-limits paths and permission/side-effect lines; the tools the
run may use; the checkable end state(s) **with their check commands**; the
bounded-evidence requirement; failure and budget exits with an escalation
trigger. Follow it with 2–3 lines of rationale listing the gaming shortcuts
you closed. `$ARGUMENTS`, when present, is the task to convert.

## Gotchas

- The `/goal` evaluator is a small model that sees **only the transcript** and
  cannot run commands — a condition that doesn't force check output into the
  transcript is unverifiable by construction.
- "Done" can be hallucinated: completion must be a predicate on state (command
  output, file diff, artifact), never the absence of further tool calls.
- Prompt-level caps are advisory. If nothing but text enforces the budget,
  say so and recommend `engineer-agent-loop` for supervisor-enforced caps.
- Don't demand evidence the run cannot produce (a check command the agent
  lacks permission to run is an infinite non-exit).
- Goal text is the task layer, not the system layer: durable identity rules
  and boundaries belong in the system prompt; the goal carries this run's
  condition, scope, and evidence standard.

## Pointers

- `references/goal-patterns.md` — goal-contract anatomy, condition checklists,
  the three exits, evidence discipline, verified `/goal` + Stop-hook
  mechanics, prompt layering.
