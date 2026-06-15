---
name: state-machine
description: Godot 4.x state machine implementation guide — enum-based and node-based patterns, state transitions, enter/exit hooks, and when to use a state machine vs simpler conditionals. Load this when implementing character states (idle/run/jump/attack), AI behavior, or any system with discrete modes that transition based on events.
---

# State Machines in Godot 4.x

## When to use a state machine

Reach for a state machine when:
- A node has mutually exclusive modes (idle, run, jump, attack, dead) with distinct behaviors per mode
- Transitions between modes have explicit conditions or trigger side effects (play animation, reset velocity)
- A growing chain of `if/elif` for a "current mode" variable is becoming hard to read or extend

For simple two-state toggles, a plain boolean is fine. State machines pay off when you have three or more distinct behaviors.

## Enum-based state machine (lightweight)

Good for single scripts with a handful of states and simple transitions.

```gdscript
class_name Player
extends CharacterBody2D

enum State { IDLE, RUN, JUMP, FALL, ATTACK }

@export var speed: float = 200.0
@export var jump_force: float = 400.0

var current_state: State = State.IDLE

@onready var anim: AnimationPlayer = $AnimationPlayer
@onready var hitbox: CollisionShape2D = $Hitbox/CollisionShape2D

func _physics_process(delta: float) -> void:
	match current_state:
		State.IDLE:
			_state_idle(delta)
		State.RUN:
			_state_run(delta)
		State.JUMP:
			_state_jump(delta)
		State.FALL:
			_state_fall(delta)
		State.ATTACK:
			_state_attack(delta)

func _transition(new_state: State) -> void:
	if new_state == current_state:
		return
	_exit_state(current_state)
	current_state = new_state
	_enter_state(new_state)

func _enter_state(state: State) -> void:
	match state:
		State.JUMP:
			velocity.y = -jump_force
			anim.play("jump")
		State.RUN:
			anim.play("run")
		State.IDLE:
			anim.play("idle")

func _exit_state(state: State) -> void:
	match state:
		State.ATTACK:
			hitbox.disabled = true

# --- per-state logic ---

func _state_idle(delta: float) -> void:
	if Input.is_action_pressed("move_right") or Input.is_action_pressed("move_left"):
		_transition(State.RUN)
	elif Input.is_action_just_pressed("jump") and is_on_floor():
		_transition(State.JUMP)

func _state_run(delta: float) -> void:
	var dir := Input.get_axis("move_left", "move_right")
	velocity.x = dir * speed
	move_and_slide()
	if dir == 0.0:
		_transition(State.IDLE)
	elif not is_on_floor():
		_transition(State.FALL)

func _state_jump(delta: float) -> void:
	velocity += get_gravity() * delta
	move_and_slide()
	if velocity.y > 0:
		_transition(State.FALL)

func _state_fall(delta: float) -> void:
	velocity += get_gravity() * delta
	move_and_slide()
	if is_on_floor():
		_transition(State.IDLE)

func _state_attack(_delta: float) -> void:
	pass  # handled by AnimationPlayer finish signal
```

**Key pattern:** `_transition()` always calls `_exit_state` then `_enter_state`, so setup and teardown logic never gets skipped regardless of which state you're coming from.

## Node-based state machine (scalable)

When states grow complex — each needing their own `_process`, `_input`, and `_physics_process` — give each state its own node. This keeps files small and states independently testable.

```
Player/
├── player.gd          ← owns the StateMachine node, passes input down
├── StateMachine/
│   ├── state_machine.gd
│   ├── Idle/
│   │   └── idle_state.gd
│   ├── Run/
│   │   └── run_state.gd
│   └── Jump/
│       └── jump_state.gd
```

**Base state class (`state_machine.gd`):**
```gdscript
class_name StateMachine
extends Node

var current_state: Node = null

func _ready() -> void:
	for child in get_children():
		child.state_machine = self
	transition(get_child(0))  # start in first state

func transition(new_state: Node) -> void:
	if current_state:
		current_state.exit()
	current_state = new_state
	current_state.enter()

func _physics_process(delta: float) -> void:
	if current_state:
		current_state.physics_update(delta)

func _process(delta: float) -> void:
	if current_state:
		current_state.update(delta)
```

**Base state script (extend this for each state):**
```gdscript
class_name State
extends Node

var state_machine: StateMachine = null

func enter() -> void: pass
func exit() -> void: pass
func update(_delta: float) -> void: pass
func physics_update(_delta: float) -> void: pass
```

**Example state (`idle_state.gd`):**
```gdscript
class_name IdleState
extends State

@onready var player: Player = $"../.."

func enter() -> void:
	player.anim.play("idle")
	player.velocity = Vector2.ZERO

func physics_update(_delta: float) -> void:
	if Input.get_axis("move_left", "move_right") != 0.0:
		state_machine.transition(state_machine.get_node("Run"))
	elif Input.is_action_just_pressed("jump"):
		state_machine.transition(state_machine.get_node("Jump"))
```

## Choosing between approaches

| | Enum-based | Node-based |
|---|---|---|
| Script count | 1 | 1 per state + StateMachine |
| Good for | Simple characters, prototypes | Complex AI, many states |
| State isolation | Logic is in one file | Each state is fully isolated |
| Testability | Harder to test states alone | Each state can be unit tested |

Start with the enum approach. Migrate to node-based when a single state's logic grows beyond ~30–40 lines or you need per-state `_input()` handling.

## Connecting state transitions to AnimationPlayer

A common pattern is to let `AnimationPlayer` drive transitions when an animation finishes:

```gdscript
func _ready() -> void:
	anim.animation_finished.connect(_on_animation_finished)

func _on_animation_finished(anim_name: StringName) -> void:
	if anim_name == &"attack":
		_transition(State.IDLE)
```

Use `StringName` literals (`&"attack"`) for animation names — they're compared by identity and avoid string allocation every frame.
