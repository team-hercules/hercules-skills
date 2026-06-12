---
name: object-pooling
description: Godot 4.x object pooling pattern — pre-instantiating and reusing nodes to avoid per-frame instantiation cost for bullets, particles, or any frequently spawned/despawned objects. Load this when profiling reveals that instantiate()/queue_free() is a performance bottleneck, typically for high-frequency projectiles, hit effects, or enemy spawners.
---

# Object Pooling (Godot 4.x)

## When to use it

Object pooling trades memory for CPU time: you pay the instantiation cost once upfront, then reuse objects instead of creating and destroying them on every spawn.

**Profile before pooling.** Use Godot's built-in profiler (**Debugger → Profiler**) to confirm that `instantiate()` / `queue_free()` calls are actually the bottleneck. For most games they won't be — pooling adds code complexity that isn't worth it if the problem doesn't exist.

Good candidates for pooling:
- Bullets fired at high rate (10+ per second)
- Hit-spark and impact VFX that appear and disappear frequently
- Enemy spawners that churn through many instances per wave

Poor candidates:
- Objects spawned rarely (once per room, once per level)
- Objects whose state is complex to reset correctly
- Anything where `queue_free()` is not a measured bottleneck

## Core idea

Instead of `instantiate()` + `queue_free()`, you:
1. Pre-instantiate N copies at startup, add them to the scene, hide/disable them.
2. When you need one, grab an inactive instance and activate it.
3. When done, deactivate (reset + hide) instead of freeing.

## Minimal bullet pool

```gdscript
# bullet_pool.gd
class_name BulletPool
extends Node

@export var bullet_scene: PackedScene
@export var pool_size: int = 50

var _pool: Array[Bullet] = []

func _ready() -> void:
    for i in pool_size:
        var bullet: Bullet = bullet_scene.instantiate()
        add_child(bullet)
        bullet.pool = self
        bullet.deactivate()
        _pool.append(bullet)

func get_bullet() -> Bullet:
    for bullet in _pool:
        if not bullet.visible:
            return bullet
    # Pool exhausted — grow it rather than fail silently
    var bullet: Bullet = bullet_scene.instantiate()
    add_child(bullet)
    bullet.pool = self
    bullet.deactivate()
    _pool.append(bullet)
    return bullet

func return_bullet(bullet: Bullet) -> void:
    bullet.deactivate()
```

```gdscript
# bullet.gd
class_name Bullet
extends Area2D

var pool: BulletPool = null
var velocity: Vector2 = Vector2.ZERO

func activate(pos: Vector2, dir: Vector2, speed: float) -> void:
    global_position = pos
    velocity = dir.normalized() * speed
    visible = true
    set_process(true)

func deactivate() -> void:
    visible = false
    set_process(false)
    velocity = Vector2.ZERO

func _process(delta: float) -> void:
    global_position += velocity * delta

func _on_body_entered(_body: Node) -> void:
    pool.return_bullet(self)
```

```gdscript
# weapon.gd — using the pool
@onready var bullet_pool: BulletPool = $BulletPool

func fire(direction: Vector2) -> void:
    var bullet = bullet_pool.get_bullet()
    bullet.activate(muzzle.global_position, direction, 600.0)
```

## Resetting state

The most common pooling bug is forgetting to reset all state when deactivating. Make `deactivate()` an exhaustive reset:

```gdscript
func deactivate() -> void:
    visible = false
    set_process(false)
    set_physics_process(false)
    velocity = Vector2.ZERO
    # Reset any signals or timers added at activate-time
    for connection in body_entered.get_connections():
        body_entered.disconnect(connection["callable"])
```

## Pool sizing

Start with a pool large enough to cover the maximum simultaneous active objects with a comfortable margin. If the pool is too small, objects will stack up and new instances get created at runtime anyway (defeating the purpose). If too large, you waste memory needlessly.

A practical approach: set the pool size to 2× the maximum you ever expect active at once, then verify with the profiler that the pool isn't growing at runtime.

## Alternative: visibility-based lookup

For very simple cases, you can skip a separate pool class and just scan children:

```gdscript
func _get_inactive_bullet() -> Bullet:
    for child in get_children():
        if not child.visible:
            return child as Bullet
    return null  # caller handles nil case
```

This is fine for small pools (≤ 20 objects). For larger pools, the linear scan itself becomes a bottleneck — use an explicit free-list array instead.

## Pooling in an Autoload

If multiple scenes need the same pool, move it to an Autoload (see `references/autoload-singleton.md`) so bullets survive scene transitions and the pool is accessible globally.
