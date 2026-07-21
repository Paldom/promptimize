---
name: engineer-agent-loop
description: Designs and diagnoses the control flow of agentic loops - pattern selection, supervisor-enforced caps and terminal states, no-progress detection, external verifier gates, context and compaction budgets, crash recovery. Use when an agent loop stalls, drifts, repeats itself, or never stops, or when building agent orchestration. Not for goal or prompt wording, ordinary code loops, or scheduled jobs.
argument-hint: <loop to design or symptom to diagnose>
---

# engineer-agent-loop

The model is a fraction of a working agent; the loop around it does the rest.
This skill designs that loop — or diagnoses one that stalls, repeats, drifts,
or never stops — by putting control flow in code: bounded, externally
verified, recoverable. Most agent failures are stopping-problem failures, and
they are silent — no exception marks a loop that is busy going nowhere.

## When NOT to use

- **The goal text is the problem** (unverifiable condition, gameable
  done-state) → `write-agent-goal`. Identify the owning layer from evidence, not
  symptom alone: premature "done" can be a verifier bug (runtime) just as a
  vague goal can loop forever (text). Goal wording problems → there;
  missing caps, weak verifiers, broken recovery → here.
- **Prompt wording** → `optimize-prompt`. **Comparing prompt variants** →
  `evaluate-prompts`.
- **Ordinary programming loops** — an infinite `for` loop or non-terminating
  recursion is a code bug, not loop engineering.
- **Scheduling** — running something every N minutes is cron/interval-runner
  territory, not loop design.
- **Compound asks** ("fix the goal AND cap the loop"): sequence them — goal
  text via `write-agent-goal`, enforcement here.

## Workflow

1. **Pick the shape by path-knowability.** Path unknown → ReAct; knowable and
   auditable → plan-and-execute with a replanner; fixed steps → hardcoded
   pipeline, no agent. Escalate to multi-agent only on an observed failure
   (context pressure, serial-latency wall, genuine heterogeneity) — and price
   it. Selection table and recipes: `references/loop-patterns.md`.
2. **Make termination supervisor-owned.** Never let the model own stopping.
   Layer independent caps in code: iteration cap, wall-clock timeout,
   token/cost budget, cycle detection. Bounds must be *effective* — cumulative
   and session-wide; a local retry cap that an outer workflow re-enters bounds
   nothing. Define explicit terminal states — SUCCESS / FAILED /
   BUDGET_EXHAUSTED / CANCELLED — each reachable, each reported, with
   cancellation that works mid-run. Map every cap to a terminal reason (a
   wall-clock timeout is a reported TIMED_OUT/FAILED, never a silent exit);
   pre-flight budget checks must reserve worst-case output and in-flight
   concurrent spend; cancellation must also clean up still-running tool
   calls.
3. **Detect no-progress deterministically — and state-aware.** Compare
   normalized tool calls against a task-state fingerprint: a repeated cycle
   with no state change is a loop; legitimate polling (results change) and a
   single backtrack are not. Tunable defaults: identical call 3× → exit with
   diagnostic; catch A→B→A oscillation; watch tokens climbing while state
   doesn't move; 3 failed fix attempts → escalate to a human. No LLM judge
   needed: `references/reliability.md`.
4. **Gate exits on external verification.** Maker–checker split with a
   fresh-context verifier, framed adversarially ("try to disprove"); the
   verifier checks criteria only — never "verify and improve". A model
   verifier is a decorrelated probabilistic check, not proof — deterministic
   checks run first and remain the gate of record; evaluate the trajectory,
   not just the final answer.
5. **Budget the context like RAM.** Target ~60–75% utilization
   (practitioner default — tune); compact at
   semantic boundaries with a pre-compaction flush to a state file; evict
   acted-on tool output instead of summarizing it; trim the tool registry.
   Full discipline: `references/context-budget.md`.
6. **Make it crash-safe.** State files are the system of record for local
   artifacts — reconstruct from git log + progress file, never chat history.
   External side effects need more: write the intent record before acting and
   reconcile on resume (a crash between a side effect and its checkpoint is
   an unknown outcome), fence concurrent resumed supervisors with leases, and
   put idempotency keys on every side-effecting call — enforced receiver-side
   or in a durable dedup layer; the key alone is a request, not a guarantee.
   Budget breach = checkpoint with partial work preserved, not a cliff.
7. **Test the loop before trusting it.** Run it against: a mocked model that
   always wants another iteration; a verifier that always rejects; a verifier
   that hangs; budget exhaustion at exact boundaries; kill-and-restart
   mid-run; forced compaction immediately before completion. Supervisor caps
   must hold against every one of them.

## Output spec

For a design: pattern + rationale, a cap table with concrete values, terminal
states, verifier gates, the context plan, and the recovery plan — plus the
step-7 test list for anything that will run unattended. For a diagnosis: the
failure family, the trace evidence, the specific missing control, and the
fix. `$ARGUMENTS`, when present, is the loop or symptom to work on.

## Gotchas

- Prompt-level caps are advisory: ~38% of confirmed infinite agentic loops
  had model-controlled stop conditions (arxiv.org/abs/2607.01641).
- A correct-looking final answer can sit on top of wrong state — trajectory
  evaluation catches what final-answer checks miss.
- Reflection without an external verifier rubber-stamps its own work; cap
  reflection rounds — the gains land in the first one or two.
- Multi-agent wins are real and expensive: the oft-quoted 90.2%
  improvement was one internal Anthropic eval and cost ~15x the tokens.
  Quote the scope and the cost with the win.
- Everything the loop reads — repo files, tool output, retrieved pages — is
  untrusted input to both maker and verifier; injection can subvert an LLM
  checker. Least-privilege tool scopes, sandboxed side effects, human
  approval for irreversible actions, and secrets kept out of traces and
  state files are part of the loop design, not extras.
- Per-step reliability compounds: 0.95^20 ≈ 36% (arithmetic, not a provider
  stat) — shorten chains and verify per step.
- Compaction silently drops exact numbers, negations, and ruled-out
  approaches — use a fixed summary schema and flush state before compacting.

## Pointers

- `references/loop-patterns.md` — pattern selection, canonical citations,
  plan-and-execute recipe, reflection discipline, multi-agent criteria, the
  Ralph prior art and its critique.
- `references/reliability.md` — caps in code, effective bounds, budget
  mechanics, no-progress detection, hooks and idempotency, verification
  gates, compounding math, incidents-to-regression-tests.
- `references/context-budget.md` — window-as-RAM, the four levers with
  correct attribution, eviction, compaction discipline, external state,
  version-gated platform facts.
