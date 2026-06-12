---
name: godot-architecture
description: Godot 4.x engine architecture reference — nodes (what they are, virtual callbacks, the scene tree), scenes (instancing, composition), resources (data vs behavior, loading, custom resources), and an overview of the three node categories: Node2D, Node3D, and Control. Load this when designing scene structure, choosing between a node and a resource, working with the scene tree, or when unfamiliar with how Godot organizes a game.
---

# Godot Architecture (Godot 4.x)

## The three pillars: Nodes, Scenes, Resources

Everything in a Godot game is built from three primitives:

- **Nodes** — the active building blocks that do things (move, draw, detect collisions, play audio)
- **Scenes** — saved trees of nodes that can be instanced anywhere, like reusable blueprints
- **Resources** — passive data containers (textures, audio, config) that nodes read and share

Understanding when each one applies is the most important architectural decision in Godot.

---

## Nodes

A Node is the smallest unit of behavior in Godot. Every visual object, physics body, UI element, camera, and audio player is a node. Nodes live in a parent/child tree — the **scene tree** — and each node runs its callbacks every frame, physics step, or in response to input.

### Virtual callbacks

Override these in GDScript to give a node behavior:

| Method | When it runs |
|---|---|
| `_init()` | When the object is constructed (before entering the tree) |
| `_enter_tree()` | When the node joins the scene tree |
| `_ready()` | After the node and all its children have entered the tree |
| `_process(delta)` | Every rendered frame; `delta` is elapsed seconds |
| `_physics_process(delta)` | Every physics tick (60×/s by default); use for movement and physics |
| `_input(event)` | Every input event |
| `_unhandled_input(event)` | Input events not consumed by the GUI |
| `_exit_tree()` | When the node leaves the scene tree |

`_ready()` is the right place to cache node references and connect signals. `_process()` is for visual/logic updates. `_physics_process()` is for anything that touches velocity, `move_and_slide()`, or collision.

### Scene tree navigation

```gdscript
# Get a child node by path
@onready var sprite: Sprite2D = $Sprite2D
@onready var hud: Control = $UI/HUD

# Add/remove children at runtime
var bullet = bullet_scene.instantiate()
add_child(bullet)
bullet.queue_free()  # safe deferred removal

# Tree-level helpers
get_tree()           # SceneTree — the root of everything
get_parent()         # this node's parent
get_children()       # Array of immediate children
is_inside_tree()     # whether the node is currently in the tree
```

### Node groups

Groups let you tag and address multiple nodes without keeping direct references:

```gdscript
# In the node's script or editor
add_to_group("enemies")

# From anywhere
get_tree().call_group("enemies", "take_damage", 10)
var enemies = get_tree().get_nodes_in_group("enemies")
```

---

## Scenes

A scene is a saved tree of nodes stored as a `.tscn` file. Think of it as a reusable blueprint — a Player scene, an Enemy scene, a Bullet scene, a MainMenu scene.

**Scenes solve the same problem as prefabs** (Unity) or blueprints (Unreal), but in Godot the concept is simpler: every scene is just a file that describes a node tree. You can instance a scene as a child of any other scene.

```gdscript
# Load and instance a scene at runtime
var bullet_scene: PackedScene = preload("res://entities/bullet.tscn")

func _on_fire_pressed() -> void:
    var bullet = bullet_scene.instantiate()
    bullet.global_position = muzzle.global_position
    get_parent().add_child(bullet)
```

### Composition over deep inheritance

Godot encourages you to build complex objects by **composing scenes**, not by creating deep class hierarchies. An Enemy is not a subclass of Character — it's a scene that contains a `CharacterBody2D`, a `HealthComponent` scene, a `HurtboxComponent` scene, and so on. Each component is its own scene with its own script and signals.

This keeps individual scripts small and lets you reuse components (health, hitbox, AI) across many different enemy types without code duplication.

### Scene inheritance

When one scene shares most of its structure with another, you can use scene inheritance: the child scene extends the parent scene and can add or override nodes and properties. Changes to the parent propagate automatically.

---

## Resources

A Resource is a **data container** — it holds information but doesn't run callbacks or live in the scene tree. Nodes use resources; resources don't use nodes.

| | Node | Resource |
|---|---|---|
| Has behavior | Yes (`_process`, `_ready`, etc.) | No |
| Lives in scene tree | Yes | No |
| Loaded in memory | Once per instance | Once, **shared** across all users |
| Saved as | `.tscn` | `.tres` / `.res` / engine formats |

The sharing behavior is important: if two `Sprite2D` nodes reference the same `Texture2D` resource, they share one copy in memory. If you load the same `.tres` file twice, Godot returns the same object. To get an independent copy, call `.duplicate()`.

### Common resource types

| Resource | What it holds |
|---|---|
| `Texture2D` | Image data for sprites and UI |
| `AudioStream` | Audio file (WAV, OGG, MP3) |
| `PackedScene` | A saved scene (`.tscn`) ready to instance |
| `Script` | A GDScript (or other language) file |
| `Material` / `ShaderMaterial` | Rendering surface properties |
| `AnimationLibrary` | Named animations for `AnimationPlayer` |
| `Font` | Font data for UI text |
| `TileSet` | Tile definitions for `TileMap` |
| `Curve` / `Gradient` | Data curves for tweens and effects |

### Loading resources

```gdscript
# preload — resolved at compile time, path must be a constant
var icon: Texture2D = preload("res://assets/icon.png")

# load — resolved at runtime, path can be a variable
var skin: Texture2D = load("res://skins/" + skin_name + ".png")
```

Use `preload` whenever the path is fixed — it catches missing files at parse time and avoids runtime overhead.

### Custom resources

Extend `Resource` to create strongly-typed data objects — equivalent to Unity's ScriptableObjects:

```gdscript
class_name EnemyData
extends Resource

@export var display_name: String = "Enemy"
@export var max_health: int = 100
@export var move_speed: float = 150.0
@export var drop_table: Array[ItemData] = []
```

Save as a `.tres` file in the editor, wire it into a node's `@export var data: EnemyData`, and all enemies of that type share the same configuration asset. Changing the `.tres` updates every instance instantly.

---

## Node categories: 2D, 3D, and Control

Godot organizes the node type system into three major branches, each inheriting from a different base:

```
Node
├── Node2D          (2D spatial nodes — CanvasItem branch)
│   ├── Sprite2D
│   ├── CharacterBody2D
│   ├── Area2D
│   ├── RigidBody2D
│   ├── StaticBody2D
│   ├── Camera2D
│   ├── AnimatedSprite2D
│   ├── TileMapLayer
│   └── AudioStreamPlayer2D
│
├── Node3D          (3D spatial nodes)
│   ├── MeshInstance3D
│   ├── CharacterBody3D
│   ├── Area3D
│   ├── RigidBody3D
│   ├── StaticBody3D
│   ├── Camera3D
│   ├── DirectionalLight3D
│   ├── OmniLight3D
│   ├── CollisionShape3D
│   └── AudioStreamPlayer3D
│
└── Control         (UI nodes — also a CanvasItem branch)
    ├── Label
    ├── Button / TextureButton
    ├── LineEdit / TextEdit
    ├── Panel / NinePatchRect
    ├── TextureRect
    ├── ProgressBar
    ├── VBoxContainer / HBoxContainer / GridContainer
    └── RichTextLabel
```

### Node2D

The base for all 2D game objects. Adds `position`, `rotation`, `scale`, and `z_index` relative to its parent. All of these have `global_*` variants for world-space values.

```gdscript
extends Node2D

func _process(delta: float) -> void:
    position.x += 100.0 * delta   # move right in local space
    rotation += 0.5 * delta       # rotate (radians)
```

Use `CharacterBody2D` for player/enemy movement with `move_and_slide()`. Use `Area2D` for trigger zones (hitboxes, collectibles). Use `RigidBody2D` when you want the physics engine to drive motion (crates, projectiles).

### Node3D

The base for all 3D game objects. Adds a `Transform3D` that encodes position, rotation (as a `Basis`), and scale in 3D space.

```gdscript
extends Node3D

func _process(delta: float) -> void:
    position += transform.basis.z * speed * delta  # move forward
    rotate_y(turn_speed * delta)
```

Use `CharacterBody3D` + `move_and_slide()` for 3D characters. Use `MeshInstance3D` to render a mesh. Use `Camera3D` to define what the player sees.

### Control

The base for all UI elements. Unlike Node2D/Node3D which position by transform, Control nodes position using an **anchor and offset** system relative to their parent container or the viewport. This lets UIs adapt to different screen sizes without manual math.

```gdscript
extends Control

func _ready() -> void:
    # anchors define which corner/edge of the parent to attach to
    anchor_right = 1.0   # stretch to fill parent width
    anchor_bottom = 1.0

func _gui_input(event: InputEvent) -> void:
    if event is InputEventMouseButton:
        print("clicked")
```

Container nodes (`VBoxContainer`, `HBoxContainer`, `GridContainer`) automatically arrange their Control children — use them instead of positioning children manually. `Theme` resources control fonts, colors, and styles across an entire UI tree from a single asset.

---

## Choosing the right structure

| Situation | Use |
|---|---|
| A thing that moves, draws, or reacts every frame | Node (Node2D, Node3D, Control) |
| Shared configuration across many instances | Custom Resource |
| A reusable group of nodes (player, enemy, UI widget) | Scene (`.tscn`) |
| Data that follows a node around at runtime | Node child or member variable |
| Data shared across many nodes (e.g., item stats) | Resource |
| Behavior that multiple unrelated nodes need | Script on a child node (component pattern) |
