# Technique Rules for Prompt Rewriting

Evidence-based rules an agent applies when rewriting a prompt. Facts verified against
primary sources as of 2026-07-18; version-tied items are tagged for re-verification.

## Contents

- [Instruction design](#instruction-design)
- [Few-shot examples](#few-shot-examples)
- [Chain-of-thought](#chain-of-thought)
- [Persona](#persona)
- [Structured output](#structured-output)
- [Decomposition](#decomposition)
- [Layered default stack](#layered-default-stack)

## Instruction design

- Put the task first: instruction at the top or in the system prompt, never buried
  under context (https://help.openai.com/en/articles/6654000-prompt-engineering).
- Exception — long-context prompts: context first, question LAST
  (https://ai.google.dev/gemini-api/docs/prompting-strategies). For prompt caching:
  static content first, variable user content last.
- Separate instructions from data with delimiters (`###` or triple quotes)
  (https://help.openai.com/en/articles/6654000-prompt-engineering).
- Specify role, audience, format, and constraints explicitly. Role + output contract
  + negative constraints outperform persona flourishes.
- Write negative constraints ("do NOT ...") with the same specificity as positive
  ones. Ban-lists are underused and effective against style/format drift.
- Precision over persona: a precise instruction with no role beats a role plus a
  vague instruction (https://www.nature.com/articles/s41598-025-19170-9).
- Remove contradictions between instructions — modern models follow instructions
  with high precision, and contradictions waste reasoning tokens. Use Markdown
  structure only where semantically correct (as of July 2026 — verify:
  https://developers.openai.com/cookbook/examples/gpt-5/gpt-5_prompting_guide).
- Repetition is a regime call, not a universal rule: deduplicate rules for
  reasoning models (single precise statements; duplicates burn reasoning
  tokens), while deliberate instruction repetition and bookend ordering
  measurably help standard models, especially on long prompts
  (https://arxiv.org/html/2512.14982v1). Don't apply both defaults at once.

## Few-shot examples

- Dosage: 2–5 diverse, high-quality examples for classification, extraction, style,
  and format tasks. Returns diminish and turn negative past 5–8; too many overfit.
- Format consistency and label distribution matter more than label correctness
  (Min et al. 2022). Inconsistent example formatting actively hurts.
- Coverage: at least 1 example per class plus 1 near-boundary case; 3 diverse
  examples beat 10 similar ones. Cover edge cases (ambiguous input, empty input),
  not just the happy path.
- Include one deliberately plain-but-correct "sanity baseline" example to stop
  drift toward verbosity and inflated tone.
- When to skip: if zero-shot already works, add nothing — examples bias output
  toward their own distribution. Vendor guidance differs: OpenAI documents
  zero-shot-first for its reasoning models; Google recommends "always include
  few-shot examples in your prompts"
  (https://ai.google.dev/gemini-api/docs/prompting-strategies).
- Selection trap: dynamic (retrieved, semantically similar) example selection is
  not reliably better than random selection (https://arxiv.org/abs/2512.22966,
  clinical-domain study). Dynamic few-shot is common for text-to-SQL/extraction —
  still A/B it against zero-shot and random selection before shipping.
- Few-shot collapse: too-similar examples teach the surface pattern, not the task.
- Handoff boundary: few-shot fails on genuine multi-step reasoning — that is where
  CoT (on non-reasoning models) takes over.
- Evidence for the technique overall: in a 5-LLM × 5-strategy comparison, few-shot
  was the only consistently positive strategy
  (https://www.nature.com/articles/s41598-025-19170-9).

## Chain-of-thought

### Regime split — where CoT pays

- Meta-analysis of 100+ papers: CoT gains concentrate in math, symbolic, and logic
  tasks; near-zero elsewhere. On MMLU, no-CoT ≈ CoT unless the question contains
  "=" (Sprague et al., "To CoT or not to CoT?", https://arxiv.org/abs/2409.12183).
- Wharton "Prompting Science Report 2: The Decreasing Value of Chain of Thought"
  (https://arxiv.org/abs/2506.07142), GPQA Diamond: non-reasoning model gains
  4.4% (GPT-4o-mini, not significant), 11.7% (Sonnet 3.5), 13.5% (Gemini Flash
  2.0). Reasoning models: 2.9% (o3-mini), 3.1% (o4-mini), −3.3% (Gemini Flash
  2.5). Time overhead: reasoning models 20–80% longer, non-reasoning models
  35–600% longer.
- Rule: prompt CoT only on non-reasoning models, and only for math/symbolic/
  multi-step tasks. Heuristic: if a human would write the steps down, CoT helps.
- CoT can hurt simple retrieval and classification (second-guessing a correct
  first impression — Gemini Flash 2.5 measured −3.3% with CoT) and
  perception-heavy multimodal tasks.
- Reasoning models: "think step by step" buys ≤~3% measured gain at 20–80%
  extra latency — a trade OpenAI explicitly advises against. OpenAI verbatim: "Avoid chain-of-thought prompts: Since these models perform
  reasoning internally, prompting them to 'think step by step' or 'explain your
  reasoning' is unnecessary." (OpenAI reasoning best-practices guide; see also
  https://openai.com/index/reasoning-models-chain-of-thought-controllability/).
  Control depth via API knobs instead — OpenAI `reasoning_effort`; Anthropic
  adaptive thinking (extended-thinking `budget_tokens` on older models only —
  it is deprecated/rejected on the newest Claude models) (as of July 2026 —
  verify: https://openai.com/index/reasoning-models-chain-of-thought-controllability/).
- Few-shot CoT exemplars add ~nothing over zero-shot CoT on strong models
  (https://arxiv.org/abs/2506.14641) — don't spend tokens on worked examples of
  reasoning.
- Long-context tasks are where CoT still earns its cost
  (https://arxiv.org/abs/2502.20790).

### Named-stages pattern (when CoT is warranted)

- Name 4–7 domain-specific reasoning stages instead of generic "think step by
  step".
- Wrap the trace in `<reasoning>`/`<answer>` tags; keep the trace out of
  user-facing output.

### Reason-then-format (two-pass)

- Format tax: schema-constrained decoding degrades reasoning on hard tasks
  ("thinking in JSON").
- Fix: pass 1 reasons in free text; pass 2 is a cheap formatting call into the
  schema.
- Anthropic: grammar constraints do not apply inside the thinking region, so
  extended thinking + structured output avoids the tax (as of July 2026 — verify:
  https://docs.anthropic.com).

## Persona

- Persona is a tone/vocabulary/framing device, not an accuracy lever. Role-based
  prompting scored WORST of five strategies across 5 LLMs
  (https://www.nature.com/articles/s41598-025-19170-9).
- Role + vague instruction underperforms a precise instruction with no role.
- Still use persona for advisory, critique, and style tasks: concrete role +
  scenario + output constraints. Persona goes in the system prompt; the task goes
  in the user turn.

## Structured output

### Mode tiers

- Four rungs, in reliability order: prompt-and-pray (never in production) →
  provider JSON mode (valid syntax only) → validate-and-retry (e.g. Instructor:
  auto-retry feeding the validation error back; raises tail latency on failing
  calls) → constrained decoding / strict schema mode (schema-conformant by
  construction, modest overhead). Retry composes with either mode — a retry
  loop on top of strict decoding covers residual semantic failures. Published
  percentages for these rungs trace only to secondary blogs — trust the
  ordering, not any specific numbers.
- JSON mode ≠ schema adherence. OpenAI verbatim: "While both ensure valid JSON is
  produced, only Structured Outputs ensure schema adherence." (as of July 2026 —
  verify: https://developers.openai.com/api/docs/guides/structured-outputs).
- OpenAI strict mode requires: all properties required, `additionalProperties:
  false`, optionality expressed via null types (as of July 2026 — verify:
  https://platform.openai.com/docs/guides/structured-outputs).
- Even native structured outputs leave a small residual failure rate
  (single-source estimate 1–5%) — keep a retry loop.
- Gemini: JSON mime type + `response_schema`, JSON Schema subset only; streamed
  chunks are valid partial JSON; Google warns structured output can reduce quality
  on tuned models (as of July 2026 — verify:
  https://ai.google.dev/gemini-api/docs/structured-output).

### Schema discipline

- Fabrication trap: a required field with absent source data invites a hallucinated
  plausible value, and retry loops convert uncertainty into confident wrong values.
- Nullable-fields discipline: make possibly-absent fields nullable or allow
  "unknown"; request evidence spans; keep schemas flat; handle refusals as a
  distinct branch; mind the output-token ceiling.
- Enum bias: constrained decoding forces SOME answer into a required enum —
  expect confident wrong values. Practitioner reports find model-emitted
  confidence scores uncorrelated with accuracy — do not rely on them to catch
  this.
- Validate values, not shape: schema-valid ≠ semantically correct. Always parse
  and validate with pydantic/zod-style checks on the values themselves.
- Field order matters: JSON schema field order changes extraction behavior
  (autoregressive generation) — place reasoning-dependent fields deliberately,
  after the fields they depend on (practitioner reports:
  https://news.ycombinator.com/item?id=48113518).
- Prefill pattern where supported: pre-fill the assistant turn with `{` (or a
  `<json>` tag) plus a stop sequence when no native mode exists. On Claude,
  prefill is incompatible with native structured outputs; forcing `tool_use` is
  the reliable JSON path there (as of July 2026 — verify:
  https://docs.anthropic.com).

## Decomposition

- Decompose only when a single prompt is unwieldy; keep cohesive tasks in one
  prompt. Cost of chaining: more calls, more latency.
- Two-step chains (diagnose → generate) often beat single mega-prompts.
- Standard shapes: plan → execute → verify, or draft → critique → revise.
  Validate intermediate outputs at each step.

## Layered default stack

Apply in order; stop at the first layer that meets the quality bar. Combinations
beat single techniques — CoT + few-shot gave the best precision/recall balance,
while self-reflection was unstable (https://arxiv.org/abs/2510.16091).

1. Baseline: role + task + explicit constraints + output format.
2. Few-shot: only for ambiguity zero-shot can't resolve (2–5 diverse examples).
3. CoT: only where the regime split says it pays (non-reasoning model,
   math/symbolic/multi-step) and proven by A/B.
4. Schema: strict structured output on machine-consumed output only, with
   nullable-fields discipline.
5. Decompose: last resort, when the single prompt stays unwieldy.
