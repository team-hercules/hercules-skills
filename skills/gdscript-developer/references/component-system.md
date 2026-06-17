---
name: component-system
description: Godot 4.x component-based design pattern — building complex game objects by composing child node components instead of deep inheritance hierarchies, with guidance on when composition beats inheritance. Load this when designing reusable behaviors like health, hitboxes, or AI, or when a class hierarchy is becoming hard to extend.
---

# Component System (Godot 4.x)

## The problem with deep inheritance

Inheritance works well for "is-a" relationships with clear hierarchies. It breaks down when:

- Multiple unrelated behaviors need to share an object (`Enemy` that can both swim and fly)
- A subclass needs most but not all of a parent's behavior
- You want to reuse one specific behavior (health, hitbox) across unrelated entity types

The classic trap: `CharacterBody2D → Character → Enemy → FlyingEnemy → FlyingBossEnemy` — changing anything deep in the chain risks breaking everything above it.

## The component approach

In Godot, a component is a plain `Node` child with its own script. It manages one responsibility and communicates outward via signals. The parent node (the entity) owns the components and wires them together — but each component is blind to its siblings and parent.

```
Player/                    ← CharacterBody2D, thin orchestrator
├── HealthComponent        ← tracks HP, emits health_changed / died
├── HurtboxComponent       ← Area2D, detects hits, tells HealthComponent
├── HitboxComponent        ← Area2D, deals damage on contact
├── MovementComponent      ← handles velocity, move_and_slide
└── AnimationComponent     ← drives AnimationPlayer based on state signals
```

Each of these can be dropped into any entity — an `Enemy`, a `Destructible`, a `Turret` — without modification.

## Implementing a component

Components expose their behaviour through signals and a clean public API. They never reach up to their parent or sideways to siblings.

```gdscript
# health_component.gd
class_name HealthComponent
extends Node

signal health_changed(new_health: int, max_health: int)
signal died

@export var max_health: int = 100

var current_health: int:
	set(value):
		current_health = clampi(value, 0, max_health)
		health_changed.emit(current_health, max_health)
		if current_health == 0:
			died.emit()

func _ready() -> void:
	current_health = max_health

func take_damage(amount: int) -> void:
	current_health -= amount

func heal(amount: int) -> void:
	current_health += amount
```

```gdscript
# hurtbox_component.gd
class_name HurtboxComponent
extends Area2D

signal hit(damage: int, hit_by: Node)

func _on_area_entered(area: Area2D) -> void:
	if area.has_method("get_damage"):
		hit.emit(area.get_damage(), area)
```

## Wiring components in the entity

The entity (parent) connects component signals together. It knows about its children; the children don't know about each other.

```gdscript
# enemy.gd
class_name Enemy
extends CharacterBody2D

@onready var health: HealthComponent = $HealthComponent
@onready var hurtbox: HurtboxComponent = $HurtboxComponent
@onready var anim: AnimationPlayer = $AnimationPlayer

func _ready() -> void:
	hurtbox.hit.connect(_on_hit)
	health.died.connect(_on_died)
	health.health_changed.connect(_on_health_changed)

func _on_hit(damage: int, _source: Node) -> void:
	health.take_damage(damage)

func _on_health_changed(current: int, maximum: int) -> void:
	# update health bar, flash sprite, etc.
	pass

func _on_died() -> void:
	anim.play("death")
	await anim.animation_finished
	queue_free()
```

Because `HealthComponent` and `HurtboxComponent` know nothing about `Enemy`, you can drop the exact same components onto a `Destructible` crate:

```gdscript
# destructible.gd
extends StaticBody2D

@onready var health: HealthComponent = $HealthComponent
@onready var hurtbox: HurtboxComponent = $HurtboxComponent

func _ready() -> void:
	hurtbox.hit.connect(func(dmg, _src): health.take_damage(dmg))
	health.died.connect(func(): queue_free())
```

## Components vs Inheritance: decision guide

| Situation | Prefer |
|---|---|
| Simple "is-a" with little variation (`Projectile → HomingProjectile`) | Inheritance |
| Reusing one behavior across unrelated types | Component |
| An entity needs multiple independent behaviors | Component |
| A subclass needs most but not all of parent behavior | Component |
| Deep hierarchy (3+ levels) becoming fragile | Refactor to components |
| Prototype / small game, iteration speed matters | Inheritance (simpler) |

A useful heuristic: if you're writing `if enemy is FlyingEnemy` more than once, inheritance is leaking across the codebase — time for components.

## Scene-based reuse

Because each component is a scene (`.tscn`), you can share them in the editor — drag `HealthComponent.tscn` into any entity's scene tree. The component's exported properties can then be tuned per-entity in the Inspector (`max_health = 50` for a goblin, `max_health = 500` for a boss) without touching the script.

## Tradeoffs

**Benefits:** High reusability, loosely coupled, easy to add behaviors without touching existing code, individually testable.

**Costs:** More files, more signal wiring, the entity becomes a coordinator with more connections to maintain. Debugging requires tracing through more indirection.

Start with inheritance for simple cases. Reach for components when a hierarchy deepens past 2–3 levels, or when you find yourself copying behavior between unrelated classes.
