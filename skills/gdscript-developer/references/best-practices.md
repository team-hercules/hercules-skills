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

See `references/signals.md` for full signal syntax.

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

The default config enforces the conventions described in this file: snake_case variables/functions, PascalCase classes/enums, CONSTANT_CASE constants, past-tense signals, max 100-character lines, tabs for indentation, and the script code order from the "Script Code Order" section above. When in doubt about a specific rule, read the `.gdlintrc` in the project — the key names map directly to what `gdlint` checks.
