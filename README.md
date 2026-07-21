# Promptimize

[![CI](https://github.com/Paldom/promptimize/actions/workflows/ci.yml/badge.svg)](https://github.com/Paldom/promptimize/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![skills.sh](https://skills.sh/b/Paldom/promptimize)](https://skills.sh/Paldom/promptimize)

Agent Skills for prompt optimization - tune prompts for the latest frontier models like Fable 5 and GPT 5.6, write effective goals and success criteria, and engineer agentic loops that converge.

Agent Skills for [Claude Code](https://code.claude.com/docs/en/skills) (and any
[Agent Skills](https://agentskills.io)-compatible tool). Each skill is a folder under
[`skills/`](skills/) with a single-purpose `SKILL.md`, trigger evals, and optional
scripts/references — validated on every write, commit, and PR.

## Quick start

Install with the [skills CLI](https://skills.sh) — auto-detects 70+ agents
(Claude Code, Codex, Cursor, Copilot, pi, …):

```bash
npx skills add Paldom/promptimize                  # all detected agents
npx skills add Paldom/promptimize -a codex -a pi   # or target specific agents
```

Or with the [GitHub CLI](https://cli.github.com/manual/gh_skill_install) (≥ 2.90),
including version-pinned installs from releases:

```bash
gh skill install Paldom/promptimize
gh skill install Paldom/promptimize <skill> --pin <tag>
```

Or as a Claude Code plugin:

```
/plugin marketplace add Paldom/promptimize
/plugin install promptimize@promptimize
```

Or copy a single skill into a project:

```bash
git clone https://github.com/Paldom/promptimize.git
cp -r promptimize/skills/<skill-name> your-project/.claude/skills/
```

Then just describe the task — the skill activates on its description — or invoke it
explicitly with `/<skill-name>`.

## Skills

| Skill | Description |
| --- | --- |
| [`optimize-prompt`](skills/optimize-prompt/) | Rewrites and tunes existing prompts for current frontier models — reasoning-vs-standard regime rules, contradiction removal, few-shot dosage, structured-output discipline, and a before/after check on every rewrite. |
| [`write-agent-goal`](skills/write-agent-goal/) | Drafts the goal text for autonomous agent runs (`/goal`, unattended sessions): machine-checkable success conditions, evidence requirements, scope and permission limits, retry and budget exits. |
| [`engineer-agent-loop`](skills/engineer-agent-loop/) | Designs and diagnoses agentic loops that converge: supervisor-enforced caps and terminal states, deterministic no-progress detection, external verifier gates, context and compaction budgets, crash recovery. |
| [`evaluate-prompts`](skills/evaluate-prompts/) | Builds honest prompt evaluations: test sets from real failures, paired A/B comparisons with sample-size math, deterministic checks first, calibrated LLM judges with bias controls. |

The four skills compose into a full improvement workflow — inventory → rewrite
prompts → harden goals → engineer loops → verify with evals.
[docs/setup-prompt.md](docs/setup-prompt.md) ships a paste-ready `/goal` that
runs it end to end over a repository.

## Repository structure

```
skills/                  # distributed skills, one folder per skill (SKILL.md + evals/ + scripts/)
docs/                    # skill-authoring guide, eval methodology, deployment guide
scripts/                 # deterministic validator used by hooks and CI
skills.sh.json           # skills.sh repo-page customization (groupings)
.claude/                 # agentic dev setup: hooks + bundled add-skill / publish-repo skills
.claude-plugin/          # plugin + marketplace manifests (makes this repo installable)
.local/                  # gitignored working area: sources, research, PROMPT.md (see below)
```

## Working on this repo with an agent

This repo is agent-native: canonical agent instructions live in
[AGENTS.md](AGENTS.md) (CLAUDE.md imports it), hooks validate every `SKILL.md` on
write, `make check` runs the full validator, and CI enforces the same gate on every
PR. The bundled `add-skill` skill walks the eval-first authoring workflow described
in [docs/skill-authoring.md](docs/skill-authoring.md). Maintainers drive sessions
with their own (gitignored, personal) `.local/PROMPT.md` goal prompt.

## Contributing

Contributions welcome — see [CONTRIBUTING.md](CONTRIBUTING.md) for the skill-proposal
process, the authoring workflow, and the PR checklist. Please note the
[Code of Conduct](CODE_OF_CONDUCT.md).

## Support

Questions, ideas, or something not working? Start with [SUPPORT.md](SUPPORT.md) —
bugs and skill proposals have [issue templates](../../issues/new/choose), and
security concerns go through [SECURITY.md](SECURITY.md) (never a public issue).

## License

[MIT](LICENSE) © 2026 Paldom
