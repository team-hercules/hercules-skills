---
name: signals
description: GDScript 4.x signals reference — declaring signals with typed parameters, emitting them, connecting with .connect()/.disconnect(), guard patterns to avoid double-connecting, and await usage. Load this when working with signals, connecting nodes, or implementing event-driven communication between nodes.
---

# Godot Signals (Godot 4.x)

## Overview

Signals are Godot's built-in pub/sub mechanism for decoupled node communication. A node emits a signal when something happens; other nodes connect to that signal and react — without either node needing a direct reference to the other.

### Common use cases

- `HealthComponent` emits `health_changed` → HUD updates the health bar
- `Hurtbox` emits `hit` → parent `Enemy` reacts to being struck
- `Button.pressed` → any node that cares about the button click

## Declaring and emitting signals

```gdscript
extends Node

signal health_changed(new_health: int)
signal player_died

func take_damage(amount: int) -> void:
    health -= amount
    health_changed.emit(health)
    if health <= 0:
        player_died.emit()
```

Signal parameters are typed in Godot 4. Emit passes values positionally to every connected callback.

## Connecting signals

```gdscript
extends Node

@onready var button: Button = $Button

func _ready() -> void:
    if not button.pressed.is_connected(_on_button_pressed):
        button.pressed.connect(_on_button_pressed)


func _on_button_pressed() -> void:
    # do something
    pass
```

The guard (`if not … is_connected`) prevents double-connecting, which would fire the callback twice per emission.

## Disconnecting

```gdscript
button.pressed.disconnect(_on_button_pressed)
```

Nodes connected via `connect()` are **not** automatically disconnected when either node is freed — you must call `disconnect()` explicitly if the lifetime of the subscriber differs from the emitter. The exception: connecting with the `CONNECT_ONE_SHOT` flag auto-disconnects after the first emission.

## One-shot and deferred connections

```gdscript
# Disconnect automatically after firing once
some_signal.connect(_on_fired, CONNECT_ONE_SHOT)

# Invoke the callback on the next frame (deferred)
some_signal.connect(_on_fired, CONNECT_DEFERRED)
```

## Awaiting signals

```gdscript
func _ready() -> void:
    await get_tree().create_timer(1.0).timeout
    print("one second later")

func wait_for_value() -> void:
    var new_health: int = await health_changed  # receives the emitted int
    print(new_health)
```

`await` suspends the current function without blocking the engine. Any other signals, physics, or rendering continues normally.

