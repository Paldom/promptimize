# Context Budget — Managing the Window in Agentic Loops

Reference for engineering agentic loops: treat the context window as a scarce,
budgeted resource. Practitioner guidance plus version-gated platform facts.
Numbers are labeled by strength — benchmark, single-source, or practitioner
convergence. Do not upgrade a hedge when reusing a claim.

## Contents

- [1. The window-as-RAM model](#1-the-window-as-ram-model)
- [2. The four levers (attribution matters)](#2-the-four-levers-attribution-matters)
- [3. Eviction before summarization](#3-eviction-before-summarization)
- [4. Compaction discipline](#4-compaction-discipline)
- [5. External state](#5-external-state)
- [6. Platform facts (version-gated)](#6-platform-facts-version-gated)

---

## 1. The window-as-RAM model

- Treat the window as RAM and external stores as disk. Every loop turn is a
  paging decision: what stays resident, what pages out to files, what gets
  reloaded on demand.
- Target 60–75% window utilization at steady state; reserve the rest for tool
  replies and output.
- Trigger compaction proactively at 70–90% full, never at hard overflow — at
  99% there is no room left to write the summary itself.
- Label honestly: the 60–75% and 70–90% figures are multi-source practitioner
  convergence, **not benchmarks**. Treat them as defaults to tune, not laws.
- Diagnose long-loop failures as context problems, not model problems:
  - poisoning — wrong facts persist and keep getting cited;
  - distraction — window too long, model drifts off task;
  - confusion — superfluous material or tools pull attention;
  - clash — contradictory facts coexist in the window.
- Recovery from all four is a context-pruning problem, not a
  smarter-model problem.

## 2. The four levers (attribution matters)

- **Write / Select / Compress / Isolate** is LangChain's taxonomy
  (https://langchain.com/blog/context-engineering-for-agents):
  - Write — externalize plans and facts to files outside the window.
  - Select — retrieve only what this turn needs.
  - Compress — summarize when under pressure.
  - Isolate — give sub-agents disposable narrow contexts; they return
    distilled summaries only, never their raw transcripts.
- Anthropic's effective-context-engineering post uses its **own** framing —
  compaction, structured note-taking, sub-agent architectures, plus
  just-in-time retrieval
  (https://anthropic.com/engineering/effective-context-engineering-for-ai-agents).
- Do not attribute write/select/compress/isolate to Anthropic. The two
  taxonomies overlap but are distinct; cite the right source for each.
- Isolate at scale, Anthropic's own data: an orchestrator-worker research
  system (Opus 4 lead + Sonnet 4 subagents) beat single-agent Opus by 90.2%
  on an internal eval, at ~15x the tokens of chat; guidance was 3–5 parallel
  subagents (https://anthropic.com/engineering/built-multi-agent-research-system).
  Isolation buys quality with tokens — budget for it.
- Isolation also fixes verification: a fresh-context verifier sub-agent
  auditing claims against the artifact + rubric only avoids inheriting the
  misreading that caused the error; self-critique in the same window does not.

## 3. Eviction before summarization

- Prefer eviction over summarization for acted-on tool output: once acted on,
  drop it and leave a one-line pointer — "ran migration, see
  logs/mig_042.txt". Deterministic, zero drift, zero LLM cost.
- Deterministic no-LLM pruning is the zero-drift option generally; summarize
  only when the budget forces it, not every turn.
- Offload oversized tool results wholesale: LangChain's deepagents write
  >20K-token tool results to the filesystem and keep only a reference in
  context (https://blog.langchain.com/context-management-for-deepagents/).
- Trim the tool registry per task and per sub-agent: every unused schema
  costs tokens on every turn and raises wrong-tool risk.
- Budget per segment, not globally. Reference split (~37K window):
  - system + tools: 4K — static; cache it
  - task framing: 1K — resident
  - retrieved material: 8K — top-5 reranked
  - tool results: 10K — evict once acted on
  - scratchpad: 6K
  - output reserve: 8K
- Scratchpad discipline: cheap deterministic per-iteration pruning plus a
  circuit breaker. Community defaults: max 10 iterations; escalate on the
  same error 3x or 5 consecutive failures; keep the 5 most recent attempts.

## 4. Compaction discipline

- Compact at semantic boundaries — a sub-task just resolved — and suppress
  compaction mid-derivation. Rubric-based triggers beat raw token thresholds
  (SelfCompact, https://arxiv.org/abs/2606.23525).
- SelfCompact numbers, hedged correctly: up to +18.1 points on math and
  +5–9 points on agentic search, at 30–70% lower per-question cost **versus a
  no-summarization baseline** — not versus fixed-interval summarization
  (against fixed-interval it matches or exceeds quality at a fraction of the
  token cost). Single v1 preprint; directional.
- Align checkpoints with compaction moments. Treat post-compaction
  re-derivation of already-known facts as a trigger bug, not model noise.
- Pre-compaction memory flush: before compaction fires, write decisions,
  constraints, and file paths to a state file (e.g. STATE.md). Enforce with a
  PreCompact hook; re-inject by re-reading the state file plus git log —
  hooks, not prompt hope.
- Fix the summary schema so structure survives: goal, decisions + rationale,
  files touched, open tasks, active constraints, error history.
- Know what summarization silently drops: exact numbers, IDs, negations
  ("do not refund"), ruled-out approaches, and in-progress state. The
  summary then becomes unauditable ground truth. Mitigate with the fixed
  schema, entity-retention spot checks, and an external archive of the raw
  trace.
- Insert the compacted summary as a user/assistant message pair, never a
  second system message (secondary source — verify). Never split a
  tool-call/result pair across the compaction cut.
- Avoid compaction thrash: do not re-read big files immediately
  post-compaction — reference the artifacts instead.

## 5. External state

- State files are the system of record. The agent is a stateless judgment
  layer: read state on boot, write back via tools. Separate short-term
  working state from long-term memory. Mandatory for 12h+ runs and
  suspend/resume.
- Reconstruct state each window from git log plus a progress file, **not**
  from chat history
  (https://anthropic.com/engineering/effective-harnesses-for-long-running-agents).
- Per-cycle skeleton that keeps state external: read memory index → do ONE
  bounded unit of work ("fix the next failing test", not "make progress") →
  spawn a fresh-context verifier → append the lesson to memory (dedupe;
  delete disproven notes) → run the stop check.
- Sliding window + anchors: keep verbatim the first message (intent), user
  corrections, and the last successful tool result; summarize or drop the
  middle.
- Order for lost-in-the-middle: system prompt + hard constraints first,
  retrieved material in the middle, current task last — attention favors
  both ends. Magnitude: GPT-3.5-Turbo on 20-doc QA fell from 75.8% (relevant
  doc in position 1) to 53.8% (middle), a >20-point drop, below its 56.1%
  closed-book score (https://arxiv.org/abs/2307.03172).
- Same principle in Anthropic's guidance: put longform data at the top,
  the query at the end — "up to 30 percent" better in their tests on
  complex multi-document inputs
  (https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices).

## 6. Platform facts (version-gated)

- Anthropic Messages API compaction: beta header `compact-2026-01-12`, edit
  type `compact_20260112`, default trigger 150,000 input tokens, minimum
  50,000 (as of July 2026 — verify:
  https://platform.claude.com/docs/en/build-with-claude/compaction).
- Claude Code auto-compaction: fires automatically near the context limit;
  there is **no published ~80% trigger figure** — the "~80%" claim in
  circulation is secondary-blog material. Tune via
  `CLAUDE_CODE_AUTO_COMPACT_WINDOW`; Sonnet 5's default threshold is ~967K
  of the 1M window (as of July 2026 — verify:
  https://code.claude.com/docs/en/model-config).
- Claude Code compaction controls: `/compact` with focus arguments and a
  documented PreCompact hook. "PostCompact" and "InstructionsLoaded" appear
  only in blogs — verify before relying on them (as of July 2026 — verify:
  https://code.claude.com/docs/en/hooks).
- Anthropic context editing (auto-clears stale tool results) + file-based
  memory tool, official stats: +39% over baseline with memory tool + context
  editing combined, +29% with context editing alone, 84% token reduction
  (as of July 2026 — verify: https://claude.com/blog/context-management).
  Ignore third-party "0.95^N compounding" framings of these stats — not in
  any Anthropic source.
- Comparable mechanisms elsewhere, for orientation (as of July 2026):
  Microsoft Agent Framework compaction — MessageGroup,
  TokenBudgetComposedStrategy (verify:
  https://learn.microsoft.com/en-us/agent-framework/agents/conversations/compaction);
  Cloudflare Session API — overlays, protectHead, tailTokenBudget (verify:
  https://developers.cloudflare.com/agents/concepts/conversation-state-and-memory/);
  Google ADK Context Compaction (verify: https://developers.googleblog.com).
