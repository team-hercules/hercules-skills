---
name: scene-management
description: Godot 4.x scene management patterns — switching scenes, additive loading, background/async loading with a loading screen, transition effects, and managing scene lifetime. Load this when implementing level transitions, a main menu, loading screens, or any system that changes what scene is active.
---

# Scene Management (Godot 4.x)

## Simple scene switching

For small games where instant transitions are acceptable:

```gdscript
# Switch by file path — Godot loads and frees the old scene
get_tree().change_scene_to_file("res://levels/level_2.tscn")

# Switch by preloaded PackedScene — faster if the scene is already in memory
var next_scene: PackedScene = preload("res://levels/level_2.tscn")
get_tree().change_scene_to_packed(next_scene)
```

Both methods block the main thread while loading. For small scenes this is fine; for large ones it causes a visible freeze — use background loading instead.

## Additive scene loading

Add a scene on top of the current one without replacing it. Useful for HUDs, pause menus, pop-ups, or streaming open-world chunks.

```gdscript
var hud_scene: PackedScene = preload("res://ui/hud.tscn")

func show_hud() -> void:
	var hud = hud_scene.instantiate()
	get_tree().root.add_child(hud)

func hide_hud(hud: Node) -> void:
	hud.queue_free()
```

## Background loading with a loading screen

For large scenes, use `ResourceLoader`'s threaded API so the game remains responsive while loading.

```gdscript
# loading_screen.gd
extends Control

var _load_path: String = ""
var _progress: Array = []  # passed by reference to get_status

func start_loading(scene_path: String) -> void:
	_load_path = scene_path
	ResourceLoader.load_threaded_request(scene_path)
	set_process(true)

func _process(_delta: float) -> void:
	var status = ResourceLoader.load_threaded_get_status(_load_path, _progress)

	match status:
		ResourceLoader.THREAD_LOAD_IN_PROGRESS:
			$ProgressBar.value = _progress[0] * 100.0

		ResourceLoader.THREAD_LOAD_LOADED:
			set_process(false)
			var packed: PackedScene = ResourceLoader.load_threaded_get(_load_path)
			get_tree().change_scene_to_packed(packed)

		ResourceLoader.THREAD_LOAD_FAILED:
			push_error("Failed to load scene: " + _load_path)
			set_process(false)
```

Trigger the loading screen from any scene via an Autoload (see [autoload-singleton.md](autoload-singleton.md)):

```gdscript
# scene_manager.gd (Autoload)
extends Node

const LOADING_SCREEN = preload("res://ui/loading_screen.tscn")

func load_scene(path: String) -> void:
	var screen = LOADING_SCREEN.instantiate()
	get_tree().root.add_child(screen)
	screen.start_loading(path)
```

```gdscript
# from anywhere
SceneManager.load_scene("res://levels/world_2.tscn")
```

## Transition effects

Fade to black before switching, then fade back in. Use a full-screen `ColorRect` with an `AnimationPlayer` or `Tween`:

```gdscript
# scene_manager.gd (Autoload, continued)
@onready var fade_rect: ColorRect = $FadeRect

func transition_to(path: String) -> void:
	# Fade out
	var tween = create_tween()
	tween.tween_property(fade_rect, "modulate:a", 1.0, 0.3)
	await tween.finished

	get_tree().change_scene_to_file(path)

	# Fade in after scene is ready
	await get_tree().process_frame
	tween = create_tween()
	tween.tween_property(fade_rect, "modulate:a", 0.0, 0.3)
```

## Persisting data across scene changes

`change_scene_to_*` frees the old scene. Anything you need to carry across — player stats, inventory, settings — must live in an Autoload (see [autoload-singleton.md](autoload-singleton.md)) or be saved to disk (see [save-system.md](save-system.md)).

## Scene lifetime and memory

Scenes loaded with `preload()` are kept in memory for the duration of the script that holds them. If you preload a large scene in a script that stays alive, that scene stays in memory.

Resources are reference-counted (there is no garbage collector) — a resource is freed the moment the last reference to it disappears. To release a preloaded resource, drop the reference:

```gdscript
var _cached_scene: PackedScene = preload("res://levels/big_level.tscn")

func unload() -> void:
	_cached_scene = null  # reference dropped; freed once nothing else references it
```

For runtime-loaded resources, Godot's resource cache may keep them alive. Call `ResourceLoader.load()` with `cache_mode = ResourceLoader.CACHE_MODE_IGNORE` if you don't want caching:

```gdscript
var fresh = ResourceLoader.load("res://path/scene.tscn", "", ResourceLoader.CACHE_MODE_IGNORE)
```

## Pause and unpause

The scene tree has a built-in pause system. Nodes with `process_mode = PROCESS_MODE_PAUSABLE` (the default) stop processing when paused:

```gdscript
# Pause everything
get_tree().paused = true

# Resume
get_tree().paused = false
```

Nodes that should keep running while paused (e.g., the pause menu itself) need `process_mode = PROCESS_MODE_ALWAYS` in the Inspector or in `_ready()`:

```gdscript
func _ready() -> void:
	process_mode = Node.PROCESS_MODE_ALWAYS
```
