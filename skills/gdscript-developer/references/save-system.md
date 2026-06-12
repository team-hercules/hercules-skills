---
name: save-system
description: Godot 4.x save system patterns — writing and reading save data with FileAccess (JSON and binary), the Persist group pattern for scene-level saves, custom Resource saves, versioning, and the user:// path. Load this when implementing save/load functionality, checkpoints, or persistent player progress.
---

# Save System (Godot 4.x)

## Save location

Always save to `user://` — Godot maps this to the OS-specific user data directory (e.g., `~/.local/share/YourGame/` on Linux, `AppData/Roaming/YourGame/` on Windows). Never save to `res://` in a shipped game (it's read-only in exports).

```gdscript
const SAVE_PATH = "user://savegame.sav"

# Check if a save exists
if FileAccess.file_exists(SAVE_PATH):
    load_game()
```

## JSON save (human-readable)

Good for save data that designers or players might inspect or mod. The tradeoff: JSON only supports primitive types — decompose `Vector2`, `Color`, etc. into floats.

### Writing

```gdscript
func save_game() -> void:
    var data = {
        "version": 1,
        "player": {
            "pos_x": player.global_position.x,
            "pos_y": player.global_position.y,
            "health": player.health,
            "inventory": player.inventory.map(func(item): return item.id),
        },
        "flags": GameState.completed_flags,
        "play_time": GameState.play_time,
    }
    var file = FileAccess.open(SAVE_PATH, FileAccess.WRITE)
    file.store_string(JSON.stringify(data, "\t"))  # "\t" for readable formatting
```

### Reading

```gdscript
func load_game() -> void:
    var file = FileAccess.open(SAVE_PATH, FileAccess.READ)
    var json = JSON.new()
    var err = json.parse(file.get_as_text())
    if err != OK:
        push_error("Save file corrupt: " + json.get_error_message())
        return

    var data: Dictionary = json.data
    player.global_position = Vector2(data["player"]["pos_x"], data["player"]["pos_y"])
    player.health = data["player"]["health"]
    GameState.completed_flags = data["flags"]
```

## Binary save (compact, typed)

Smaller files, faster I/O, and handles most Godot types (`Vector2`, `Vector3`, `Color`, `Rect2`, etc.) natively. Use when save files are large or performance matters.

```gdscript
func save_game() -> void:
    var file = FileAccess.open(SAVE_PATH, FileAccess.WRITE)
    file.store_var({
        "version": 1,
        "position": player.global_position,  # Vector2 — works natively
        "health": player.health,
        "inventory": player.inventory_ids,
    })

func load_game() -> void:
    var file = FileAccess.open(SAVE_PATH, FileAccess.READ)
    var data: Dictionary = file.get_var()
    player.global_position = data["position"]
    player.health = data["health"]
```

`store_var` / `get_var` use Godot's internal serialization format. Do not mix this with JSON files.

## Persist group pattern (scene-level saves)

When an entire scene's state needs saving — objects collected, doors opened, enemies killed — use the Persist group to automatically collect all saveable nodes.

Add nodes that need saving to the `"Persist"` group (in the editor or via `add_to_group("Persist")`). Each one implements a `save()` method:

```gdscript
# chest.gd
extends Node2D

var opened: bool = false

func save() -> Dictionary:
    return {
        "scene": scene_file_path,
        "parent": get_parent().get_path(),
        "pos_x": global_position.x,
        "pos_y": global_position.y,
        "opened": opened,
    }
```

Save all Persist nodes:

```gdscript
func save_scene() -> void:
    var file = FileAccess.open(SAVE_PATH, FileAccess.WRITE)
    for node in get_tree().get_nodes_in_group("Persist"):
        if not node.has_method("save"):
            continue
        file.store_line(JSON.stringify(node.save()))
```

Load them back:

```gdscript
func load_scene() -> void:
    # Clear existing Persist nodes to avoid duplicates
    for node in get_tree().get_nodes_in_group("Persist"):
        node.queue_free()

    if not FileAccess.file_exists(SAVE_PATH):
        return

    var file = FileAccess.open(SAVE_PATH, FileAccess.READ)
    var json = JSON.new()
    while file.get_position() < file.get_length():
        json.parse(file.get_line())
        var data: Dictionary = json.data
        var obj = load(data["scene"]).instantiate()
        get_node(data["parent"]).add_child(obj)
        obj.global_position = Vector2(data["pos_x"], data["pos_y"])
        obj.opened = data["opened"]
```

## Saving custom Resources

If your save data maps naturally to a `Resource` (see `references/resource-data.md`), you can save and load it directly:

```gdscript
class_name SaveData
extends Resource

@export var player_position: Vector2
@export var health: int = 100
@export var inventory: Array[String] = []
@export var completed_levels: Array[int] = []

func write() -> void:
    ResourceSaver.save(self, "user://savegame.tres")

static func read() -> SaveData:
    if not FileAccess.file_exists("user://savegame.tres"):
        return SaveData.new()
    return load("user://savegame.tres") as SaveData
```

`.tres` files are human-readable text resources — easy to inspect during development. Use `.res` (binary) for shipped games if you want to discourage manual editing.

## Save versioning

Add a `version` field from day one. When the save format changes, increment it and migrate old saves:

```gdscript
const CURRENT_VERSION = 2

func load_game() -> void:
    var data = _read_raw()
    var version = data.get("version", 1)

    if version < 2:
        data = _migrate_v1_to_v2(data)

    _apply(data)

func _migrate_v1_to_v2(old: Dictionary) -> Dictionary:
    # v2 renamed "hp" → "health"
    old["health"] = old.get("hp", 100)
    old.erase("hp")
    old["version"] = 2
    return old
```

## Multiple save slots

Parameterise the save path:

```gdscript
func save_path(slot: int) -> String:
    return "user://save_slot_%d.sav" % slot

func save(slot: int) -> void:
    var file = FileAccess.open(save_path(slot), FileAccess.WRITE)
    # ...

func get_save_slots() -> Array[int]:
    var slots: Array[int] = []
    for i in range(1, 4):
        if FileAccess.file_exists(save_path(i)):
            slots.append(i)
    return slots
```

## Encryption (optional)

For anti-cheat or protecting sensitive data, use `FileAccessEncrypted`:

```gdscript
const KEY = "your-secret-key-32-chars-exactly"

func save_encrypted(data: Dictionary) -> void:
    var file = FileAccessEncrypted.new()
    file.open_encrypted_with_pass("user://save.enc", FileAccess.WRITE, KEY)
    file.store_string(JSON.stringify(data))
    file.close()
```

Note: the key is embedded in the binary — this is obfuscation, not true security.
