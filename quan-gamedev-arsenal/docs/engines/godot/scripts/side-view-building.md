# Building System Controller (Godot 4.5)

Quan Hoang’s top-down / platformer building system controller for Godot 4.5.  
This script allows players to preview and place blocks or tools in the world, with optional UI integration and safety checks.

---

## Module Overview

**Root Node:** `Node2D` (Player reference optional)  
**Engine:** Godot 4.5  
**Script Type:** Building/Placement Controller

This module handles all logic related to build mode, tool previews, snapping to a grid, tool switching, and optional UI feedback.

---

## Features

- Toggle build mode on/off
- Tool/block preview (ghost preview)
- Snap-to-grid placement
- Tool switching (cycle through available tools/blocks)
- Optional UI feedback via `Label` and `AnimatedSprite2D`
- Fully modular and inspector-friendly

---

## Prerequisites

### Scene Structure

Main Scene (Node2D)

├── Player (Node2D) (optional, for player reference)

├── UI (CanvasLayer) (optional)

│ ├── Label (mode display) ← assign to 'label_mode'

│ └── AnimatedSprite2D ← assign to 'ui_build'

Blocks/Tools Scenes

├── A.tscn

├── B.tscn

└── C.tscn

**Notes:**

- Player node is optional but recommended for player-relative building.
- UI nodes (`Label` and `AnimatedSprite2D`) are optional and safely ignored if not assigned.
- Blocks/tools must be individual scenes that can be instanced.

---

## Input Map

The following input actions must be defined in  
**Project Settings → Input Map**:

| Action         | Expected Key / Button        |
| -------------- | ---------------------------- |
| `toggle_build` | Key to toggle build mode     |
| `switch_tool`  | Key to cycle through tools   |
| `left_click`   | Mouse button or key to place |

---

## Script

```gdscript

## ============================================================
##  QUAN HOANG BUILDING SYSTEM CONTROLLER (GODOT 4.5)
## ============================================================
##
##  This script handles a top-down / platformer style building
##  system where the player can preview and place blocks/tools
##  in the world.
##
##  Features:
##  - Build mode toggle
##  - Tool/block preview (ghost)
##  - Snap to grid placement
##  - Tool switching
##  - Optional UI feedback (label + AnimatedSprite2D)
##  - Optional integration with wire manager or other nodes
##  - Fully safe: optional nodes checked before usage
##
##  ------------------------------------------------------------
##  NODE TREE PREREQUISITES
##  ------------------------------------------------------------
##
##  Main Scene (Node2D)
##  ├── Player (Node2D)                 ← optional, used for player reference
##  ├── UI (CanvasLayer)                 ← optional
##  │    ├── Label (mode display)        ← assign to 'label_mode'
##  │    └── AnimatedSprite2D           ← assign to 'ui_build'
##
##  Blocks/Tools Scenes
##  ├── A.tscn
##  ├── B.tscn
##  └── C.tscn
##
##  ------------------------------------------------------------
##  INPUT MAP REQUIREMENTS
##  ------------------------------------------------------------
##   - "toggle_build" → key to toggle build mode
##   - "switch_tool"  → key to cycle through blocks/tools
##   - "left_click"   → key/mouse button to place tool
##
## ============================================================

extends Node2D

# ============================================================
# PLAYER SETTINGS
# ============================================================
@export_group("Player References")

@export var player: Node2D                         # Optional: reference to player for positioning
@export var place_range: float = 200.0             # Max placement distance from player

# ============================================================
# GRID SETTINGS
# ============================================================
@export_group("Grid Settings")

@export var grid_size: int = 32
@export var grid_offset: Vector2 = Vector2(0, -16)

# ============================================================
# BUILD MODE SETTINGS
# ============================================================
@export_group("Build Mode Settings")

@export var toggle_build_action: String = "toggle_build"
@export var next_block_action: String = "switch_tool"
const PLACE_TILE_ACTION := "left_click"

var build_mode: bool = true
var ghost: Node2D = null
var current_block_index: int = 0
var placed_tools: Dictionary = {}

# ============================================================
# UI REFERENCES (OPTIONAL)
# ============================================================
@export_group("UI")

@export var label_mode: Label
@export var ui_build: AnimatedSprite2D

# ============================================================
# BLOCKS / TOOLS
# ============================================================
@export_group("Blocks / Tools")

@export var blocks := [
	{
		"name": "A",
		"scene": preload("res://Scenes/A.tscn")
	},
	{
		"name": "B",
		"scene": preload("res://Scenes/B.tscn")
	},
	{
		"name": "C",
		"scene": preload("res://Scenes/C.tscn")
	}
]

# ============================================================
# OTHER NODE REFERENCES
# ============================================================
@export_group("Other Nodes")

@export var wire_manager: Node2D                 # Optional: integrate with wiring system


# ============================================================
# GODOT LIFECYCLE
# ============================================================

func _process(_delta: float) -> void:
	# Redraw the ghost / preview every frame
	queue_redraw()


func _physics_process(_delta: float) -> void:
	preview_tool_place()


func _input(event) -> void:
	# Toggle build mode
	if event.is_action_pressed(toggle_build_action):
		_toggle_build_mode()
		return

	if not build_mode:
		return

	# Place tool
	if event.is_action_pressed(PLACE_TILE_ACTION):
		place_tool_at_mouse()

	# Switch selected block/tool
	if event.is_action_pressed(next_block_action):
		switch_block()


# ============================================================
# BUILD MODE MANAGEMENT
# ============================================================

## Toggles build mode on/off.
func _toggle_build_mode() -> void:
	build_mode = !build_mode
	if label_mode:
		label_mode.visible = build_mode
	print("Build mode:", build_mode)

	# Remove ghost when exiting build mode
	if not build_mode and ghost:
		ghost.queue_free()
		ghost = null


# ============================================================
# TOOL/BLOCK SELECTION
# ============================================================

## Switches to the next block/tool in the array.
func switch_block() -> void:
	current_block_index = (current_block_index + 1) % blocks.size()
	var name = blocks[current_block_index]["name"]
	print("Selected block:", name)

	if ui_build:
		ui_build.animation = name

	if ghost:
		ghost.queue_free()
		ghost = null


# ============================================================
# TOOL/BLOCK PREVIEW (GHOST)
# ============================================================

## Shows a semi-transparent ghost at mouse position, snapped to grid.
func preview_tool_place() -> void:
	if not build_mode:
		return

	# Create ghost if it doesn't exist
	if ghost == null:
		ghost = blocks[current_block_index]["scene"].instantiate()
		_make_ghost(ghost)
		get_tree().current_scene.add_child(ghost)

	# Move ghost to snapped mouse position
	var mouse_pos = get_global_mouse_position()
	ghost.global_position = snap_to_grid(mouse_pos)


## Converts a position to grid-aligned coordinates.
func snap_to_grid(pos: Vector2) -> Vector2:
	var p = pos - grid_offset
	return Vector2(
		round(p.x / grid_size) * grid_size,
		round(p.y / grid_size) * grid_size
	) + grid_offset


## Lowers opacity of all Sprites/AnimatedSprites and disables collisions recursively.
func _make_ghost(node: Node) -> void:
	for child in node.get_children():
		if child is Sprite2D or child is AnimatedSprite2D:
			child.modulate.a = 0.5
		elif child is Node:
			_make_ghost(child)

	if node is CollisionObject2D:
		node.collision_layer = 0
		node.collision_mask = 0


# ============================================================
# PLACE TOOL
# ============================================================

## Instantiates and places the currently selected tool at mouse position.
func place_tool_at_mouse() -> void:
	if not build_mode:
		return

	var mouse_pos = get_global_mouse_position()
	var new_tool = blocks[current_block_index]["scene"].instantiate()
	new_tool.global_position = snap_to_grid(mouse_pos)

	if "is_preview" in new_tool:
		new_tool.is_preview = false

	get_tree().current_scene.add_child(new_tool)
	print("Placed tool:", blocks[current_block_index]["name"])


```
