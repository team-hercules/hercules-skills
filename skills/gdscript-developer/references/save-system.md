---
name: save-system
description: Godot 4.x save system patterns — Resource-based saves (preferred), the Persist group pattern for scene-level saves, versioning, and the user:// path. Load this when implementing save/load functionality, checkpoints, or persistent player progress.
---

# Save System (Godot 4.x)

## Save location

Always save to `user://` — Godot maps this to the OS-specific user data directory (e.g., `~/.local/share/YourGame/` on Linux, `AppData/Roaming/YourGame/` on Windows). Never save to `res://` in a shipped game (it's read-only in exports).

```gdscript
const SAVE_PATH = "user://savegame.tres"

if FileAccess.file_exists(SAVE_PATH):
	load_game()
```

## Resource-based save (preferred)

Define a `Resource` class for your save data. Godot serializes `@export` properties automatically — no manual packing or unpacking, full type support including `Vector2`, `Color`, nested Resources, and typed arrays.

```gdscript
# save_data.gd
class_name SaveData
extends Resource

@export var version: int = 1
@export var player_position: Vector2
@export var health: int = 100
@export var inventory: Array[String] = []
@export var completed_levels: Array[int] = []
@export var play_time: float = 0.0
```

```gdscript
# game.gd
const SAVE_PATH = "user://savegame.tres"

func save_game() -> void:
	var save_data := SaveData.new()
	save_data.player_position = player.global_position
	save_data.health = player.health
	save_data.inventory = player.inventory.duplicate()
	save_data.completed_levels = game_state.completed_levels.duplicate()
	save_data.play_time = game_state.play_time
	ResourceSaver.save(save_data, SAVE_PATH)

func load_game() -> void:
	if not FileAccess.file_exists(SAVE_PATH):
		return
	var save_data := load(SAVE_PATH) as SaveData
	if save_data == null:
		push_error("Save file corrupt or wrong type")
		return
	player.global_position = save_data.player_position
	player.health = save_data.health
	player.inventory = save_data.inventory.duplicate()
	game_state.completed_levels = save_data.completed_levels.duplicate()
	game_state.play_time = save_data.play_time
```

`.tres` files are human-readable text resources — easy to inspect during development. Switch to `.res` (binary) for shipped games if you want smaller files or to discourage manual editing:

```gdscript
ResourceSaver.save(save_data, "user://savegame.res")
```

## Nested Resources

Split large save data into sub-Resources for clarity:

```gdscript
# player_save_data.gd
class_name PlayerSaveData
extends Resource

@export var position: Vector2
@export var health: int = 100
@export var level: int = 1
```

```gdscript
# save_data.gd
class_name SaveData
extends Resource

@export var version: int = 1
@export var player: PlayerSaveData
@export var completed_levels: Array[int] = []
@export var play_time: float = 0.0
```

Godot serializes nested Resources in the same `.tres` file — no extra file management needed.

## Persist group pattern (scene-level saves)

When an entire scene's state needs saving — objects collected, doors opened, enemies killed — use the Persist group to automatically collect all saveable nodes.

Each saveable node defines its own state Resource and implements `save_state()` / `load_state()`:

```gdscript
# chest_state.gd
class_name ChestState
extends Resource

@export var scene_path: String
@export var parent_path: NodePath
@export var position: Vector2
@export var opened: bool = false
```

```gdscript
# chest.gd
extends Node2D

var opened: bool = false

func _ready() -> void:
	add_to_group("Persist")

func save_state() -> ChestState:
	var state := ChestState.new()
	state.scene_path = scene_file_path
	state.parent_path = get_parent().get_path()
	state.position = global_position
	state.opened = opened
	return state

func load_state(state: ChestState) -> void:
	global_position = state.position
	opened = state.opened
```

Collect all states into a container Resource and save it:

```gdscript
# scene_save_data.gd
class_name SceneSaveData
extends Resource

@export var node_states: Array[Resource] = []
```

```gdscript
# world.gd
const SCENE_SAVE_PATH = "user://scene.tres"

func save_scene() -> void:
	var scene_save := SceneSaveData.new()
	for node in get_tree().get_nodes_in_group("Persist"):
		if node.has_method("save_state"):
			scene_save.node_states.append(node.save_state())
	ResourceSaver.save(scene_save, SCENE_SAVE_PATH)

func load_scene() -> void:
	for node in get_tree().get_nodes_in_group("Persist"):
		node.queue_free()

	if not FileAccess.file_exists(SCENE_SAVE_PATH):
		return

	var scene_save := load(SCENE_SAVE_PATH) as SceneSaveData
	if scene_save == null:
		return

	for state in scene_save.node_states:
		var node := load(state.scene_path).instantiate() as Node
		get_node(state.parent_path).add_child(node)
		if node.has_method("load_state"):
			node.load_state(state)
```

## Save versioning

Always add a `version` export to your `SaveData` Resource from day one — even if you never need it, adding it later means existing saves won't have the field and you're already in migration territory.

When a save format changes, increment `CURRENT_VERSION` and write a migration function for each old version:

```gdscript
# save_data.gd
class_name SaveData
extends Resource

@export var version: int = 1
@export var health: int = 100
@export var movement_speed: float = 200.0
# v2 added: @export var stamina: int = 100
```

```gdscript
# game.gd
const CURRENT_VERSION = 2

func load_game() -> void:
	var save_data := load(SAVE_PATH) as SaveData
	if save_data == null:
		return
	_migrate(save_data)
	_apply(save_data)

func _migrate(save_data: SaveData) -> void:
	if save_data.version < 2:
		_migrate_v1_to_v2(save_data)
	# chain further migrations as versions grow

func _migrate_v1_to_v2(save_data: SaveData) -> void:
	save_data.stamina = 100  # new field, give old saves the default
	save_data.version = 2
```

Because `SaveData` is a typed Resource, migrations are property assignments — no string key lookups, no silent failures from a typo in a key name. The compiler tells you when a field doesn't exist.

## Multiple save slots

Parameterise the save path:

```gdscript
func _save_path(slot: int) -> String:
	return "user://save_slot_%d.tres" % slot

func save(slot: int) -> void:
	var save_data := _build_save_data()
	ResourceSaver.save(save_data, _save_path(slot))

func get_filled_slots() -> Array[int]:
	var slots: Array[int] = []
	for slot_index in range(1, 4):
		if FileAccess.file_exists(_save_path(slot_index)):
			slots.append(slot_index)
	return slots
```

## Why not JSON

JSON is not recommended for save files in Godot games. The main problems:

**No Godot types.** JSON only knows numbers, strings, booleans, arrays, and objects. Every `Vector2`, `Color`, `Rect2`, `NodePath`, or enum value must be manually decomposed on write and reconstructed on read. This is boilerplate that grows with every field and breaks silently if you miss a step.

**No type safety.** A JSON load returns an untyped `Dictionary`. Every field access is a string key lookup with no compiler help. A typo like `data["helath"]` returns `null` at runtime with no error — your player silently starts with 0 health.

**Versioning is manual and fragile.** With a Resource, adding a new `@export` field with a default value is enough — old saves load fine and the new field gets its default. With JSON, you must remember to call `.get("new_field", default)` everywhere, and if you forget, you get `null` instead of the default.

**Migrations are string-key surgery.** Renaming a field in a Resource is a find-and-replace in GDScript; the type system flags any missed call site. Renaming a JSON key means writing migration code that manipulates string-keyed dictionaries — easy to get wrong, impossible to verify statically.

Use JSON only when save data genuinely needs to be human-edited outside the game (modding, level editors, external tools), or when you're interoperating with a non-Godot system that consumes the file.

```gdscript
func save_json() -> void:
	var data := {
		"version": 1,
		"position_x": player.global_position.x,
		"position_y": player.global_position.y,
		"health": player.health,
	}
	var file := FileAccess.open("user://save.json", FileAccess.WRITE)
	file.store_string(JSON.stringify(data, "\t"))

func load_json() -> void:
	var file := FileAccess.open("user://save.json", FileAccess.READ)
	var json := JSON.new()
	if json.parse(file.get_as_text()) != OK:
		push_error("Save corrupt: " + json.get_error_message())
		return
	var data: Dictionary = json.data
	player.global_position = Vector2(data["position_x"], data["position_y"])
	player.health = data["health"]
```

## Encryption (optional)

For anti-cheat or protecting sensitive data, use `FileAccessEncrypted`. Note that this wraps `FileAccess` and works with string content, so you'd write a serialized form of your Resource rather than using `ResourceSaver` directly:

```gdscript
const ENCRYPTION_KEY = "your-secret-key-32-chars-exactly"

func save_encrypted(save_data: SaveData) -> void:
	var file := FileAccessEncrypted.new()
	file.open_encrypted_with_pass("user://save.enc", FileAccess.WRITE, ENCRYPTION_KEY)
	file.store_var(inst_to_dict(save_data))
	file.close()
```

The key is embedded in the binary — this is obfuscation, not true security.
