---
name: syntax-and-features
description: Complete GDScript 4.x language reference covering variables, types, functions, classes, control flow, annotations, signals, async/await, and built-in operators. Load this when writing new scripts, working with types, using annotations like @export/@onready, or when you need to check exact GDScript syntax.
---

# GDScript Syntax and Features (Godot 4.x)

**Contents:** Variables and Constants · Built-in Types · Functions · Properties · Classes · Control Flow · Annotations · Signals · Async/Await · Type Casting · Enums · Operators · Special Keywords · Comments and Regions

## Variables and Constants

```gdscript
var a           # null by default
var b = 5
var c: int      # typed, defaults to 0
var d := "hi"  # type inferred from right-hand side

const MAX_SPEED = 200
const PI_VAL: float = 3.14

static var shared_count = 0  # belongs to the class, shared across all instances
```

Use `:=` when the type is obvious from context; use an explicit type annotation when it isn't.

## Built-in Types

**Primitives:** `null`, `bool`, `int` (64-bit), `float` (64-bit double), `String`, `StringName` (`&"name"`), `NodePath` (`^"Node/Label"`)

**Math types:** `Vector2`, `Vector2i`, `Vector3`, `Vector3i`, `Rect2`, `Transform2D`, `Transform3D`, `Quaternion`, `Basis`, `Plane`, `AABB`, `Color`

**Containers:**
```gdscript
var arr: Array = [1, 2, 3]
var typed_arr: Array[int] = [1, 2, 3]
var dict: Dictionary = {"key": "value"}
var typed_dict: Dictionary[String, int] = {}
```

**String literals:**
```gdscript
var s = "hello\nworld"
var raw = r"C:\path\no\escapes"
var multi = """multi
line"""
```

**Number literals:**
```gdscript
var hex = 0xFF
var bin = 0b1010
var big = 1_000_000
```

## Functions

```gdscript
func greet(name: String) -> String:
	return "Hello, " + name

func no_return() -> void:
	print("done")

func with_default(x: int, y: int = 0) -> int:
	return x + y

static func helper(a: int, b: int) -> int:
	return a + b  # no access to self or instance members
```

**Lambda functions:**
```gdscript
var double = func(x: int) -> int: return x * 2
print(double.call(5))  # 10
```

**Calling a parent method:**
```gdscript
func _ready() -> void:
	super()         # call parent _ready
	super._ready()  # explicit form
```

## Properties (Getters / Setters)

```gdscript
var _health: int = 100
var health: int:
	get:
		return _health
	set(value):
		_health = clamp(value, 0, max_health)
		health_changed.emit(_health)
```

## Classes

```gdscript
@icon("res://icon.png")
class_name MyClass
extends Node

## Optional doc comment describing this class.

signal something_happened

const LIMIT = 50
var value: int = 0

func _ready() -> void:
	pass
```

**Inner classes:**
```gdscript
class Projectile:
	var speed: float = 300.0
	func launch() -> void:
		pass

var p = Projectile.new()
```

**Inheritance:**
```gdscript
class_name Enemy
extends CharacterBody2D

func _init() -> void:
	super()  # call parent constructor
```

**Abstract classes (4.5+):**
```gdscript
@abstract
class_name Shape

@abstract func area() -> float
```

## Control Flow

**Conditionals:**
```gdscript
if hp <= 0:
	die()
elif hp < 20:
	play_hurt_sound()
else:
	move()

var label = "low" if hp < 20 else "ok"  # ternary
```

**Loops:**
```gdscript
for i in range(10):
	print(i)

for item in my_array:
	print(item)

for key in my_dict:
	print(my_dict[key])

while condition:
	if should_break: break
	if should_skip: continue
```

**Match (pattern matching):**
```gdscript
match state:
	"idle":
		handle_idle()
	"run", "walk":
		handle_movement()
	[var x, var y]:          # array pattern
		print(x, y)
	{"name": var n}:         # dict pattern
		print(n)
	_:
		pass                 # default / wildcard

# Pattern guards
match point:
	[var x, var y] when y == x:
		print("on diagonal")
```

## Annotations

| Annotation | Purpose |
|---|---|
| `@export` | Expose variable to Godot Inspector |
| `@export_range(min, max)` | Inspector slider |
| `@export_group("Name")` | Inspector section grouping |
| `@onready` | Initialize variable at `_ready()` time |
| `@static_unload` | Unload class static data when no instances remain |
| `@tool` | Run script in the editor |
| `@warning_ignore("code")` | Suppress a specific warning |
| `@icon("res://…")` | Set class icon in editor |
| `@abstract` | Mark class or method as abstract (4.5+) |

```gdscript
@export var speed: float = 200.0
@export_range(1, 10) var level: int = 1
@export_group("Combat")
@export var damage: int = 10

@onready var sprite: Sprite2D = $Sprite2D
@onready var label: Label = $UI/Label
```

## Signals

```gdscript
signal health_changed(new_health: int)
signal player_died

func take_damage(amount: int) -> void:
	health -= amount
	health_changed.emit(health)
	if health <= 0:
		player_died.emit()
```

See [signals.md](signals.md) for connecting and using signals.

## Async / Await

```gdscript
func _ready() -> void:
	await get_tree().create_timer(1.0).timeout
	print("one second later")

func wait_for_signal() -> void:
	var new_health = await health_changed  # receives emitted value
	print(new_health)
```

`await` suspends the current function without blocking the engine. It can await any signal or any coroutine that yields a value.

## Type Casting

```gdscript
var node = get_node("Child") as Sprite2D   # null if incompatible
var n: int = some_float as int             # forced conversion
if node is Sprite2D:
	node.flip_h = true
```

## Enums

```gdscript
enum Direction { NORTH, SOUTH, EAST, WEST }
enum State { IDLE = 0, RUN = 1, JUMP = 5 }

var current: State = State.IDLE
print(State.keys())   # ["IDLE", "RUN", "JUMP"]
print(State.values()) # [0, 1, 5]

# Unnamed enum — members become global-scoped constants in this file
enum { TILE_FLOOR, TILE_WALL }
```

## Operators

| Category | Operators |
|---|---|
| Arithmetic | `+` `-` `*` `/` `%` `**` |
| Bitwise | `~` `&` `\|` `^` `<<` `>>` |
| Comparison | `==` `!=` `<` `>` `<=` `>=` |
| Logical (prefer English form) | `and` `or` `not` |
| Membership | `in` `not in` |
| Type check | `is` `is not` |
| Compound assignment | `+=` `-=` `*=` `/=` `**=` `%=` `&=` `\|=` `^=` `<<=` `>>=` |

`/` performs integer division when both operands are `int` — `5 / 2 == 2`. Type one operand as `float` (or multiply by `1.0`) for fractional results.

## Special Keywords

| Keyword | Meaning |
|---|---|
| `self` | Current instance |
| `super` | Parent class |
| `null` | Null reference (Object-derived types only) |
| `pass` | Empty statement placeholder |
| `assert(cond)` | Debug-only assertion; stripped in release |
| `preload("res://…")` | Load resource at compile time |
| `load("res://…")` | Load resource at runtime |
| `PI`, `TAU`, `INF`, `NAN` | Built-in math constants |

## Comments and Regions

```gdscript
# Regular comment

## Documentation comment — appears in editor tooltips and docs

#region My Section
var grouped_var = 1
#endregion
```
