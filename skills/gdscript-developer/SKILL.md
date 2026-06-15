---
name: gdscript-developer
description: End-to-end skill for GDScript and Godot 4 game development. Use this skill whenever the user is writing, debugging, reviewing, or architecting GDScript code; designing Godot scenes or systems; asking about Godot-specific patterns like signals, state machines, save systems, scene management, or node composition; or building any feature inside the Godot Engine. Covers syntax, best practices, design patterns, and Godot architecture.
---

# GDScript Developer

This skill guides GDScript development in Godot 4. When a task involves writing or reviewing GDScript code, read the relevant references below and apply them — don't rely solely on general programming knowledge, as Godot has strong idioms and conventions that differ from other engines.

## How to approach a task

1. **Understand the context first.** Is the user starting fresh, mid-feature, or debugging? Do they have an existing scene structure? Understanding where they are determines which references are useful.

2. **Consult the right references before writing code.** The references cover Godot 4's idioms in depth — syntax, architecture decisions, design patterns. Check the description of each relevant file and read it before generating code in that domain.

3. **Write idiomatic Godot 4 code.** Use static typing, `@onready`, signals for decoupling, and the patterns described in the references. Code that works in a generic engine context may still be wrong for Godot.

4. **Iterate with the user.** For larger features, propose the structure (which nodes, which scenes, which patterns) before diving into implementation. Catch design issues early.

## Resources

Read the frontmatter of each reference to decide relevance; read the body for the detail you need. Don't load all references upfront — pick the ones that apply to the current task.

**Engine fundamentals**
- [godot-architecture](references/godot-architecture.md): Nodes, scenes, resources, Node2D / Node3D / Control. Start here when designing scene structure or choosing between a node and a resource.
- [syntax-and-features](references/syntax-and-features.md): Complete GDScript 4.x language reference — variables, types, functions, classes, annotations, signals, await, enums, operators.
- [best-practices](references/best-practices.md): Static typing, node access patterns, signals-over-references, performance tips, linting with `.gdlintrc`.
- [signals](references/signals.md): Declaring, emitting, connecting, disconnecting, one-shot/deferred, and await.

**Design patterns**
- [autoload-singleton](references/autoload-singleton.md): Global services across scene changes — game state, event bus, audio manager, settings.
- [resource-data](references/resource-data.md): Custom Resources as typed data containers. Dictionary conventions: prefer enums as keys; use the enum → value-object pattern.
- [component-system](references/component-system.md): Child-node components (HealthComponent, HurtboxComponent) vs inheritance — decision guide and full examples.
- [state-machine](references/state-machine.md): Enum-based and node-based state machine patterns with transition hooks.
- [object-pooling](references/object-pooling.md): Pre-instantiate and reuse nodes for high-frequency spawning. Profile first.
- [scene-management](references/scene-management.md): Scene switching, additive loading, background loading with progress, transitions, and pause.
- [save-system](references/save-system.md): JSON and binary saves, Persist group pattern, versioning, multiple slots, encryption.
