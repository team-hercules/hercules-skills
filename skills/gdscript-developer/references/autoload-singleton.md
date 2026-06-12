---
name: autoload-singleton
description: Godot 4.x Autoload singleton pattern — registration, access, common use cases (global state, event bus, audio manager), and pitfalls. Load this when you need persistent cross-scene state, a global event bus, or any system that must survive scene changes.
---

# Autoload Singleton Pattern (Godot 4.x)

## What it is

An Autoload is a node or script that Godot loads at startup and keeps alive for the entire lifetime of the game — across every scene change. Every other script can access it by name, making Autoloads the standard way to share state and services that don't belong to any single scene.

## Registration

In the editor: **Project → Project Settings → Globals → Autoload**

- Browse to your script (e.g., `res://autoloads/game_state.gd`)
- Give it a name (e.g., `GameState`) — this becomes the global identifier
- Leave **Enable** checked (this makes it directly accessible as `GameState` in any script)
- Load order is top-to-bottom; put foundational autoloads (event bus, settings) above those that depend on them

## Accessing an Autoload

```gdscript
# From anywhere — no import needed
GameState.score += 10
AudioManager.play_sfx("hit")
EventBus.player_died.emit()
```

## Structuring an Autoload

Autoloads are plain `Node` scripts. Keep them focused on one responsibility.

```gdscript
# autoloads/game_state.gd
extends Node

signal score_changed(new_score: int)

var score: int = 0:
    set(value):
        score = value
        score_changed.emit(score)

var current_level: int = 1
var lives: int = 3

func reset() -> void:
    score = 0
    current_level = 1
    lives = 3
```

## Common use cases

### Global game state

Anything that must survive a scene change — score, lives, current level, player inventory, settings.

```gdscript
# game_state.gd
extends Node
var score: int = 0
var player_health: int = 100
```

### Event bus

An Autoload that only holds signals acts as a decoupled message broker. Nodes emit events; other nodes react — without either side holding a direct reference to the other.

```gdscript
# event_bus.gd
extends Node

signal player_died
signal enemy_killed(enemy_type: String, position: Vector2)
signal level_completed(level_id: int)
```

```gdscript
# emitter (enemy.gd)
EventBus.enemy_killed.emit("goblin", global_position)

# listener (hud.gd)
func _ready() -> void:
    EventBus.enemy_killed.connect(_on_enemy_killed)

func _on_enemy_killed(enemy_type: String, _pos: Vector2) -> void:
    kill_count += 1
```

### Audio manager

Centralises sound playback so any node can trigger audio without managing AudioStreamPlayer nodes itself.

```gdscript
# audio_manager.gd
extends Node

@onready var sfx_player: AudioStreamPlayer = $SFXPlayer
@onready var music_player: AudioStreamPlayer = $MusicPlayer

var sfx_library: Dictionary = {
    "hit": preload("res://audio/hit.ogg"),
    "jump": preload("res://audio/jump.ogg"),
}

func play_sfx(key: String) -> void:
    if sfx_library.has(key):
        sfx_player.stream = sfx_library[key]
        sfx_player.play()

func play_music(stream: AudioStream) -> void:
    if music_player.stream == stream:
        return
    music_player.stream = stream
    music_player.play()
```

### Settings / config

```gdscript
# settings.gd
extends Node

const SAVE_PATH = "user://settings.cfg"

var master_volume: float = 1.0
var fullscreen: bool = false

func save() -> void:
    var cfg = ConfigFile.new()
    cfg.set_value("audio", "master_volume", master_volume)
    cfg.set_value("display", "fullscreen", fullscreen)
    cfg.save(SAVE_PATH)

func load_settings() -> void:
    var cfg = ConfigFile.new()
    if cfg.load(SAVE_PATH) != OK:
        return
    master_volume = cfg.get_value("audio", "master_volume", 1.0)
    fullscreen = cfg.get_value("display", "fullscreen", false)
```

## Pitfalls

**Never call `free()` or `queue_free()` on an autoload.** The engine manages their lifetime — freeing one manually causes crashes.

**Use `call_deferred()` when switching scenes** from inside an autoload, so the transition happens safely after the current frame completes:

```gdscript
func goto_scene(path: String) -> void:
    _do_goto.call_deferred(path)

func _do_goto(path: String) -> void:
    get_tree().change_scene_to_file(path)
```

**Don't overload Autoloads with unrelated state.** One bloated `Global.gd` becomes a maintenance nightmare. Split by responsibility: `GameState`, `AudioManager`, `EventBus`, `Settings`.

**Autoloads create global coupling** — any script can mutate them from anywhere. Prefer signals and local state for anything that doesn't genuinely need to be global. If a node only needs data from a parent, pass it via `@export` or method arguments instead.
