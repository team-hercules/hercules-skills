---
name: resource-data
description: Godot 4.x Resource-based data pattern — custom Resources as typed data containers (equivalent to ScriptableObjects), exporting them via Inspector, sharing vs duplicating, and when to use a Resource instead of a Dictionary or Autoload. Load this when defining item stats, character configs, level data, or any data that should be authored in the editor and shared across multiple nodes.
---

# Resource-Based Data Pattern (Godot 4.x)

## What it is

A custom `Resource` is a GDScript class that extends `Resource` instead of `Node`. It holds data — exported properties that can be edited in the Inspector, saved as `.tres` assets, and shared across any number of nodes. Think of it as Godot's equivalent of Unity's ScriptableObjects.

The key difference from a Dictionary or plain variable: it has a type, a schema, editor support, and automatic serialization.

## Defining a custom Resource

```gdscript
# resources/item_data.gd
class_name ItemData
extends Resource

@export var display_name: String = ""
@export var icon: Texture2D
@export var max_stack: int = 1
@export var value: int = 0
@export var description: String = ""
@export var tags: Array[StringName] = []
```

Create instances in the editor (**FileSystem → right-click → New Resource → ItemData**) or in code:

```gdscript
var sword = ItemData.new()
sword.display_name = "Iron Sword"
sword.value = 50
ResourceSaver.save(sword, "res://data/items/iron_sword.tres")
```

## Using a Resource in a node

```gdscript
# item_pickup.gd
class_name ItemPickup
extends Area2D

@export var data: ItemData  # wire up in the Inspector

func _ready() -> void:
    $Label.text = data.display_name
    $Sprite2D.texture = data.icon
```

Every instance of `ItemPickup` references the same `ItemData` asset — changing the asset updates every pickup in the game instantly.

## Sharing vs duplicating

Resources are shared by default. Two nodes pointing at the same `.tres` asset see the same object in memory:

```gdscript
# Both enemies share one EnemyData — changing one changes both
var goblin_a: EnemyData = preload("res://data/goblin.tres")
var goblin_b: EnemyData = preload("res://data/goblin.tres")
# goblin_a == goblin_b → true
```

When you need per-instance data (e.g., a character's *current* health vs. its *base* health), duplicate before modifying:

```gdscript
# character.gd
@export var base_stats: CharacterStats

var _stats: CharacterStats  # instance-specific copy

func _ready() -> void:
    _stats = base_stats.duplicate(true)  # deep duplicate
    _stats.current_health = _stats.max_health
```

`duplicate(true)` recursively duplicates sub-resources. Without `true`, nested resources are still shared.

## Common patterns

### Typed item / character databases

```gdscript
# resources/character_stats.gd
class_name CharacterStats
extends Resource

@export var max_health: int = 100
@export var attack: int = 10
@export var defense: int = 5
@export var move_speed: float = 200.0
@export var level: int = 1
```

Each enemy type gets its own `.tres` file. The scene just has `@export var stats: CharacterStats` and the designer fills it in per-enemy type — no code changes needed for balance tuning.

### Dialogue and narrative data

```gdscript
class_name DialogueLine
extends Resource

@export var speaker: String = ""
@export var text: String = ""
@export var portrait: Texture2D
@export var next_line: DialogueLine  # can chain resources
```

### Level/wave configuration

```gdscript
class_name WaveData
extends Resource

@export var enemy_types: Array[PackedScene] = []
@export var spawn_count: int = 10
@export var spawn_interval: float = 1.5
@export var boss_wave: bool = false
```

### Ability / skill definitions

```gdscript
class_name AbilityData
extends Resource

@export var ability_name: String = ""
@export var cooldown: float = 1.0
@export var damage: int = 0
@export var effect_scene: PackedScene
@export var icon: Texture2D
```

## Resources vs Dictionaries vs Autoloads

| | Custom Resource | Dictionary | Autoload |
|---|---|---|---|
| Editor support | Yes (Inspector) | No | Partial |
| Typed | Yes | No | Depends |
| Shared across scenes | Yes (as asset) | No | Yes |
| Per-instance copy | `.duplicate()` | Manual | Not applicable |
| Saved to disk | `.tres` / `.res` | Manual (JSON etc.) | Manual |
| Best for | Authored, reusable data | Dynamic/runtime data | Global services/state |

Use a **Resource** when data is authored at design time and shared across instances.
Use a **Dictionary** for transient runtime data you don't need to serialize or type-check.
Use an **Autoload** for services or state that needs to be globally accessible and mutable at runtime.

## Dictionary conventions

Prefer typed Resources, arrays, or variables over Dictionaries whenever the data shape is known at design time. Dictionaries are untyped and give up static analysis, autocomplete, and `gdlint` checks — reach for them only when the structure genuinely varies at runtime (e.g., serialised save data, API responses, dynamic config loaded from JSON).

When you do use a Dictionary, use an **enum as the key** rather than a plain string or integer. Enum keys are compile-time constants — typos become parse errors instead of silent bugs, and autocomplete works.

### Pattern: enum key + typed value object

When a dictionary maps an enum to a structured object, define all three pieces in order:

```gdscript
# 1. Enum — the set of valid keys
enum Element { FIRE, WATER, EARTH, AIR }

# 2. Value type — a class or inner class describing each entry's shape
class ElementProperties:
    var damage_multiplier: float
    var status_effect: String
    var color: Color

    func _init(mult: float, effect: String, col: Color) -> void:
        damage_multiplier = mult
        status_effect = effect
        color = col

# 3. Dictionary — enum → value object
var element_table: Dictionary = {
    Element.FIRE:  ElementProperties.new(1.5, "burn",  Color.RED),
    Element.WATER: ElementProperties.new(1.0, "wet",   Color.BLUE),
    Element.EARTH: ElementProperties.new(0.8, "slow",  Color.GREEN),
    Element.AIR:   ElementProperties.new(1.2, "none",  Color.WHITE),
}
```

Usage:

```gdscript
func get_multiplier(element: Element) -> float:
    return element_table[element].damage_multiplier
```

If the value object is simple enough to not warrant a class, a typed inner struct or a plain Resource also works — the key rule is that **the enum defines all valid keys** and **the value shape is consistent**.

### When a Dictionary is still the right call

- Runtime-dynamic keys (player-defined slot names, localisation keys from a file)
- Serialisation intermediary (JSON round-trip)
- Sparse mappings where most entries don't exist and checking `.has()` is intentional

For anything else — especially configuration data — a custom Resource with `@export` fields is almost always cleaner.

## Loading Resources in code

```gdscript
# At compile time (path is a constant) — preferred
var item: ItemData = preload("res://data/items/sword.tres")

# At runtime (path is dynamic)
var item: ItemData = load("res://data/items/" + item_id + ".tres") as ItemData
```

Preloaded resources are cached — calling `preload` on the same path twice returns the same object.
