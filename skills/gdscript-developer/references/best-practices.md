---
name: best-practices
description: GDScript 4.x coding standards and best practices covering static typing guidelines, node access patterns, signals-over-references, performance tips, export variables, and error handling. Naming conventions and code order are enforced by .gdlintrc — see the Linting section. Load this when writing new scripts, reviewing code quality, or deciding how to structure a scene or system.
---

# GDScript Best Practices (Godot 4.x)

Naming conventions and script code order are enforced by the project's `.gdlintrc` — see the **Linting** section at the bottom of this file.

## Static Typing

Always prefer typed code. Types catch bugs at edit time, enable autocomplete, and document intent.

```gdscript
# Preferred
var speed: float = 200.0
func heal(amount: int) -> void:
	_health = min(_health + amount, MAX_HEALTH)

# Use := when the type is obvious from the right-hand side
var direction := Vector2.RIGHT
var label := $UI/Label as Label  # cast makes the type clear

# Always type get_node() results — the return type is Node, not the subtype
@onready var health_bar: ProgressBar = $UI/HealthBar
```

Avoid `as` casts in hot paths — they imply a runtime type check. Use them at setup time (`_ready`) or when you genuinely need to narrow a type.

## Node Access Patterns

Use `@onready` for any node reference that's constant for the lifetime of the scene:

```gdscript
@onready var timer: Timer = $Timer
@onready var hurtbox: Area2D = $Hurtbox
```

Avoid `get_node()` in `_process()` — it traverses the scene tree every call. Cache references in `_ready()` instead.

For cross-scene communication, prefer signals over direct node references. Direct references couple nodes together and break easily when scenes are restructured.

## Signals over Direct References

Prefer signals for one-to-many notification and for decoupling child nodes from parents:

```gdscript
# health_component.gd — doesn't know who's listening
signal health_changed(new_health: int)

func take_damage(amount: int) -> void:
	health -= amount
	health_changed.emit(health)
```

Connect in the parent or scene root, not in the emitting node:
```gdscript
# player.gd
func _ready() -> void:
	health_component.health_changed.connect(_on_health_changed)

func _on_health_changed(new_health: int) -> void:
	hud.update_health(new_health)
```

Always guard connections to avoid double-connecting:
```gdscript
if not button.pressed.is_connected(_on_button_pressed):
	button.pressed.connect(_on_button_pressed)
```

See [signals.md](signals.md) for full signal syntax.

## Script Size and Responsibility

Keep scripts focused on a single responsibility. When a script exceeds ~200 lines, consider whether it's doing too much. Common splits:

- `HealthComponent` — health tracking, damage, death signal
- `MovementComponent` — velocity, physics, ground detection
- `StateMachine` — state transitions and delegation

Inner classes are useful for small data structures that belong to one script but would be overkill as separate files.

## Performance Tips

- Cache node references and expensive lookups in `_ready()`, never in `_process()`.
- Use `_physics_process()` for movement and physics; use `_process()` only for visual updates.
- Prefer `StringName` (`&"string"`) for frequently compared strings (signal names, animation names, group names) — they compare by identity, not content.
- Avoid `get_node()`, `find_child()`, or dictionary lookups inside per-frame loops.
- Use typed arrays (`Array[int]`) instead of untyped arrays when the element type is fixed — the engine can optimize them better.
- `preload()` resources at compile time when the path is known; use `load()` only when the path is dynamic.

## Export Variables

Use `@export` to expose tunable values to the Inspector rather than hardcoding them. This lets designers iterate without touching code:

```gdscript
@export var speed: float = 200.0
@export_range(0.0, 1.0, 0.05) var friction: float = 0.2
@export var jump_curve: Curve
@export_group("Audio")
@export var hit_sound: AudioStream
```

## Error Handling

Use `assert()` freely during development to catch invalid states — it's stripped from release builds:

```gdscript
assert(amount > 0, "Damage must be positive")
```

For production paths, use early returns and emit signals rather than crashing:

```gdscript
func take_damage(amount: int) -> void:
	if amount <= 0:
		return
	_health -= amount
	health_changed.emit(_health)
```

## Naming Variables

Use full, descriptive names. Short names hide intent and force the reader to guess.

```gdscript
# Bad — what is "dir"? a directory? a direction enum? a Vector2?
var dir := Vector2.RIGHT
var t := Tile.new()
var weighted: Dictionary[int, float] = {}

# Good
var movement_direction := Vector2.RIGHT
var tile := Tile.new()
var tile_weights: Dictionary[int, float] = {}
```

Common offenders to avoid:

| Shortcut | Use instead |
|---|---|
| `dir` | `direction`, `directory` (whichever applies) |
| `t`, `n`, `i` (outside loops) | the type or role: `tile`, `node`, `index` |
| `pos` | `position` |
| `vel` | `velocity` |
| `rot` | `rotation` |
| `btn` | `button` |
| `mgr` | `manager` |

For typed dictionaries, name both sides of the mapping so it's clear what maps to what:

```gdscript
# Bad — what is "weighted"?
var weighted: Dictionary[StringName, float] = {}

# Good
var action_weights: Dictionary[StringName, float] = {}
var tile_spawn_weights: Dictionary[int, float] = {}
```

Loop variables are the one exception — `i`, `j` specifically are fine for tight index loops when the intent is obvious from context. (if there is a need for 3 or more index variables inside loops - like triple nested loops, there is some bad design going on)

## Linting

All formatting and style rules are enforced by `gdlint` (part of [gdtoolkit](https://github.com/Scony/godot-gdscript-toolkit)). The active config file is `.gdlintrc` at the project root.

**Use the project's own `.gdlintrc` if one exists. If the codebase has no `.gdlintrc`, fall back to the skill's default:**
`skills/gdscript-developer/assets/default.gdlintrc`

Copy it to the project root before running the linter:

```bash
# Install gdtoolkit (once)
pip install gdtoolkit

# Lint the project
gdlint path/to/your/script.gd

# Or lint everything
gdlint **/*.gd
```

The default config enforces snake_case for variables/functions/signals, PascalCase for classes/enums, CONSTANT_CASE for constants, max 100-character lines, tab indentation, and the order of class members (the `class-definitions-order` key). When in doubt about a specific rule, read the `.gdlintrc` in the project — the key names map directly to what `gdlint` checks.

Two conventions the linter cannot verify, so keep them in mind while writing: name signals in past tense (`door_opened`, not `open_door`), and prefix private members with `_`.
