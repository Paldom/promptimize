# Setup prompt: repo-wide prompt & loop improvement

A paste-ready `/goal` that composes this repo's four skills into one
verified improvement pass over a target repository.

**How to use:** install the skills (see the README quick start), open a
Claude Code session **in the repository you want to improve**, paste the
block below as one message, and let it run. The goal keeps the session
working until every clause of "Done when" holds. Skills normally activate on
their descriptions; the prompt invokes them by `/name` explicitly so routing
is deterministic (recommended for scripted/headless runs too).

Constraints honored by design: ordering (inventory → prompts → goals →
loops → verification), parallel subagents only on disjoint file surfaces,
a verifier gate on every accepted change, and zero git actions — the
working tree is the handoff.

```text
/goal Improve every LLM prompt, agent goal, and agent loop in this repository using the promptimize skills, leaving every edit uncommitted for my review. NEVER run git commit or git push.

Method (keep this order; spawn parallel subagents ONLY for items in different files):
1. Inventory first, read-only: list every LLM prompt (system/task prompts in code or config), every autonomous-run goal text (/goal payloads, run configs), and every agent loop (harness, retry, or orchestration code), each with its file path. Post the inventory as a table BEFORE any edit.
2. Prompts: for each inventoried prompt, invoke /optimize-prompt with the prompt and its known failures. Follow its full workflow: identify the model regime first, subtract contradictions before adding anything, move prose to API knobs, and produce its change table.
3. Goals: for each autonomous-run goal text, invoke /write-agent-goal. Every produced goal must contain a check command whose output lands in the transcript, do-not-touch constraints, and retry + budget exits. Reject any draft that fails the skill's read-back step.
4. Loops: for each loop, invoke /engineer-agent-loop. Require supervisor-owned termination (layered caps in code; terminal states SUCCESS / FAILED / BUDGET_EXHAUSTED / CANCELLED), deterministic no-progress detection, an external verifier gate, and a context plan. Write out its loop-test list for anything that runs unattended.
5. Verification gate, bracketing every accepted change: re-run the failing cases on old vs new (the /optimize-prompt check step). Where stakes are high or no test set exists, invoke /evaluate-prompts to design the paired comparison first, and report the effect with its detection limits. A change with no check is not accepted - revert it.
6. Consistency pass: re-read every edited file; confirm goals name real commands and paths from this repo, loops reference real entry points, and no secrets leaked into prompts or expected transcripts.

Done when ALL of the following hold, with evidence visible in the transcript:
- the inventory table was posted before the first edit
- every inventoried prompt has an accepted, checked rewrite or a written skip reason
- every goal text passes read-back: checkable end state + named check command + scope constraints + retry/budget exits
- every loop has supervisor caps, terminal states, an external verifier gate, and its test list written
- every accepted change shows its before/after check output in the transcript
- git status shows only intended modified files, and zero commits or pushes were made
- a final summary lists each changed file with its verification result

Budget: after 3 consecutive failed verification attempts on the same item, or when the session nears its practical turn budget, stop thrashing - checkpoint partial work with a NEXT-STEPS note and park.
```

The block is ~2.7k characters — inside the community-reported ~4k `/goal`
condition budget, with room for repo-specific additions (e.g. extra
off-limits paths).

**Fact-check trail** (commands vs the shipped skills): `/optimize-prompt`
check step = its workflow step 6; `/write-agent-goal` read-back = its step 7;
`/engineer-agent-loop` terminal states and test list = its steps 2 and 7;
`/evaluate-prompts` paired-comparison design = its steps 4–6.
