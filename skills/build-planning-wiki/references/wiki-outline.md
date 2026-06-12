# Planning Wiki Outline

Use this outline when creating a new `wiki/` folder or substantially restructuring an existing one. Rename sections to match the project domain.

## File Set

- `index.md`: master index, current implementation status, page map, design pillars, and next recommended work.
- `00_tasks.md`: prioritized task board grouped by architecture, quality, features, content, tests, and release readiness.
- `01_core_workflow.md`: primary workflow, lifecycle phases, and important transitions.
- `02_data_models_and_content.md`: schemas, resource formats, catalogs, balance tables, examples, and naming rules.
- `03_systems_architecture.md`: major components, ownership boundaries, dependency flow, events/signals, persistence, and integration points.
- `04_operational_behaviors.md`: dynamic rules such as scheduling, generation, routing, policy decisions, scaling, or background work.
- `05_implementation_blueprint.md`: practical implementation plan, validation commands, migration notes, and risks.

## Page Template

```markdown
---
name: Short Human Title
description: One sentence describing what this page specifies.
---

# Short Human Title

## Current Status

| Area | Status | Notes |
|------|--------|-------|
| Example system | Partial | State what exists and what is missing. |

## Design Intent

Explain the purpose of the system and the outcome it supports.

## Specifications

Document concrete rules, data contracts, formulas, events, or lifecycle steps.

## Implementation Notes

Name the files, artifacts, resources, workflows, services, or commands affected by this plan.

## Open Tasks

- [ ] Write a concrete task with a visible completion condition.
```

## Index Requirements

The index should answer:

- What is this project?
- What phase is it in?
- Which systems are complete, partial, missing, or risky?
- Which wiki page owns each major subject?
- What are the global design or architecture pillars?
- What should an agent or developer read first before changing core systems?

## Task Page Requirements

The task page should:

- Put critical correctness and architecture work before expansion work.
- Separate quality work, system work, feature work, content work, and verification.
- Use checkboxes for work items.
- Include direct references to affected files or systems when known.
- Avoid tasks that cannot be verified.

## Maintenance Checklist

When implementation changes:

- Update status tables and checkboxes.
- Update data schemas or formulas if contracts changed.
- Add or remove index links when wiki pages move.
- Record new validation commands or known limitations.
- Keep speculative roadmap items separate from implemented facts.
