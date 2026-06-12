---
name: build-planning-wiki
description: Create or update a concise project planning wiki for any project domain. Use when Codex needs to turn project requirements, existing docs, architecture notes, process notes, task lists, or project findings into a structured `wiki/` folder with a master index, numbered topic files, implementation tasks, system specifications, design decisions, and maintenance guidance. Also use before making structural, architectural, data-contract, workflow, or major implementation decisions so the agent consults the existing wiki first and updates it after the decision.
---

# Build Planning Wiki

## Core Workflow

1. Inspect existing project context before writing:
   - Read `AGENTS.md`, `CLAUDE.md`, or equivalent agent instructions.
   - If a `wiki/` folder exists, read `wiki/index.md`, `wiki/00_tasks.md`, and enough topic files to match its naming, tone, and detail level.
   - Inspect project files only as needed to verify architecture, implemented systems, file names, and task status.

2. Build or update the wiki as a planning artifact, not a marketing document:
   - Prefer `wiki/index.md` as the entry point.
   - Use numbered topic files such as `00_tasks.md`, `01_core_loop.md`, `02_data_model.md`, and `03_technical_architecture.md`.
   - Keep each file focused on one planning surface: tasks, state machines, data models, content catalogs, technical architecture, risks, or validation.
   - Include YAML frontmatter with `name` and `description` in every wiki page when the existing wiki uses it.

3. Ground the plan in the actual project:
   - Separate implemented behavior from proposed behavior.
   - Name real files, artifacts, resources, workflows, commands, and configuration when known.
   - Mark uncertainty explicitly instead of presenting guesses as facts.
   - Preserve project terminology and existing architecture decisions unless the user asks for a redesign.

4. Keep tasks actionable:
   - Write checklists with concrete outcomes, not vague intentions.
   - Group tasks by priority or subsystem.
   - Include validation commands and acceptance criteria when they are known for the project.

5. Maintain the wiki after changes:
   - Update the relevant wiki pages when implementation changes architecture, task status, data contracts, balance rules, or workflow.
   - Add new pages only when a subject is large enough to deserve its own file.
   - Update `wiki/index.md` links whenever adding, removing, or renaming wiki files.

## Architectural Decision Workflow

Use the wiki as the project's architectural memory.

Before making a structural, architectural, data-contract, workflow, or major implementation decision:

1. Read `wiki/index.md` first when it exists.
2. Read the topic page that owns the affected system or workflow.
3. Check `wiki/00_tasks.md` or the equivalent planning page for related open work, risks, and sequencing.
4. If the wiki is missing the relevant topic, create or update the smallest useful page before proceeding with the decision.

After making the decision:

- Update the canonical wiki page with the chosen approach, affected files or artifacts, constraints, validation steps, and any meaningful tradeoffs.
- Record follow-up work in the task page when the decision creates implementation, migration, testing, or documentation work.
- Keep rejected alternatives brief; include them only when they clarify future maintenance.
- Update `wiki/index.md` if the decision adds, removes, renames, or changes ownership of wiki pages.

## Recommended Structure

For a new planning wiki, start from `references/wiki-outline.md` and adapt the file names to the project. Load that reference only when creating a new wiki or doing a major restructuring.

For a small update to an existing wiki, do not load the outline unless needed. Match the current wiki style and edit the minimum set of pages required.

## Writing Rules

- Use concise Markdown with clear headings, tables, formulas, and checklists where they improve scanning.
- Use status labels such as `Complete`, `Partial`, `Missing`, `Planned`, or `Blocked` consistently.
- Prefer exact implementation notes over broad advice.
- Do not duplicate the same system description across multiple pages; link to the canonical page instead.
- Do not create extra docs such as README, changelog, or process notes inside the skill output unless the user explicitly asks.
