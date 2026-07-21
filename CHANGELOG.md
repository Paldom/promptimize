# Changelog

All notable changes to this repository's skills are documented here.
Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/);
versioning: [SemVer](https://semver.org) on the plugin manifest
(breaking skill-interface change → major, new skill → minor, fix → patch).

## [Unreleased]

### Added
- `optimize-prompt` — rewrites and tunes existing prompts for current frontier
  models (regime rules, contradiction removal, few-shot dosage, structured
  output, checked rewrites), with references on techniques, model-specific
  notes (Claude Fable 5, GPT-5.6), and automated-optimization escalation.
- `write-agent-goal` — drafts goal text for autonomous agent runs with
  machine-checkable success conditions, evidence requirements, scope and
  permission limits, and retry/budget exits; verified `/goal` + Stop-hook
  mechanics in references.
- `engineer-agent-loop` — designs and diagnoses agentic loops: supervisor-owned
  termination, effective bounds, no-progress detection, external verifier
  gates, context/compaction budgeting, crash recovery.
- `evaluate-prompts` — standalone prompt evaluation: failure-derived test sets,
  paired A/B with sample-size math, deterministic checks before calibrated
  LLM judges.
- `docs/setup-prompt.md` — paste-ready `/goal` composing the four skills into
  a repo-wide improvement workflow.
- Repository scaffolded from the skills template.
