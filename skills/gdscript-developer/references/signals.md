---
name: signals
description: Overview on godot signals how and when to use them. Use this when you need to understand how signals work, how they are called and when to use them in godot based games. 
---

# Godot signals

## Overview

Signals in godot are a mechanism which allows us to make nodes watch each other for events and trigger
Much like a publisher subscriber design pattern but with nodes instead

### Common usecases

- Player health updated and the health bar needs to be notified - the HealthComponent emits a health_changed signal which the health bar (or parent node bound to it) is connected to it and changes the contents displayed in the health bar
- Hurtbox got hit and needs to notify that it was hit

## Syntax

### Declaring signals

```gdscript
extends Node

signal some_signal(some_parameter: String)


func some_function():
	# logic
	some_signal.emit("some value")
```

### Using signals

```gdscript
extends Node

@onready button: Button = $Button

func _ready():
	if !button.pressed.is_connected(_on_button_pressed):
		button.pressed,connect(_on_button_pressed)


func _on_button_pressed():
	# do something
```

