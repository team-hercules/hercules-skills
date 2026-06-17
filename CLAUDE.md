# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A library of agent skills (hercules-skills). There is no application code, build step, or test suite — the deliverables are skill directories made of Markdown instructions and bundled resources.

Two distinct skill locations:

- `skills/` — the skills authored in this repo (the product). Currently `gdscript-developer` and `build-planning-wiki`.
- `.agents/skills/skill-creator/` — the tooling skill used to create, evaluate, and improve the skills in `skills/`. Do not edit it as part of authoring work; use it.

## How to work here (from AGENTS.md)

- Understand the user's prompt deeply, and write skills cooperatively with the user.
- Use the `skill-creator` skill in `.agents/skills/skill-creator/` for creating and improving skills.
- Use its `analyzer`, `comparator`, and `grader` agents (in `.agents/skills/skill-creator/agents/`) to validate skill quality.

## Skill anatomy

Each skill is a directory containing:

- `SKILL.md` (required) — YAML frontmatter with `name` and `description` (the description is the triggering mechanism and should be written to be a bit "pushy"), followed by Markdown instructions. Keep under ~500 lines; push detail into references.
- `references/` — docs loaded into context on demand; SKILL.md should say when to read each one.
- `agents/` — instructions for specialized subagents.
- `scripts/` / `assets/` — executable helpers and output templates (optional).

## Evaluation tooling (run from `.agents/skills/skill-creator/`)

The skill-creator's SKILL.md documents the full create → test → review → improve loop. Key commands:

```bash
# Validate a skill's structure
python -m scripts.quick_validate <path/to/skill>

# Aggregate eval run results into benchmark.json/.md
python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>

# Open the human review viewer for an iteration's outputs
python eval-viewer/generate_review.py <workspace>/iteration-N --skill-name <name> --benchmark <workspace>/iteration-N/benchmark.json

# Optimize a skill's frontmatter description for triggering accuracy
python -m scripts.run_loop --eval-set <trigger-eval.json> --skill-path <path> --model <model-id> --max-iterations 5 --verbose

# Package a finished skill into a .skill file
python -m scripts.package_skill <path/to/skill-folder>
```

Eval workspaces live in `<skill-name>-workspace/` as a sibling of the skill directory, organized as `iteration-N/eval-N/{with_skill,without_skill}/outputs/`. Test prompts live in the skill's `evals/evals.json` (schema in `.agents/skills/skill-creator/references/schemas.md`).
