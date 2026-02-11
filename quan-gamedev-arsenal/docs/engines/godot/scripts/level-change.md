# Level Changer (Godot 4.5)

A simple reusable script to transition the player to the next level when entering an `Area2D`.

---

## Module Overview

**Root Node:** `Area2D`  
**Engine:** Godot 4.5

This script is designed to be attached to an `Area2D` node with a collision shape. When a body (ideally the player) enters the area, the script calculates the next level number and loads the corresponding scene.

---

## Features

- Detects when the player enters the `Area2D`
- Automatically increments the level number and loads the next scene
- Optional safety check to ensure the next level scene exists
- Can be configured to only respond to nodes in the "Player" group

---

## Prerequisites

### Scene Naming Convention

- Levels must be named consistently as:  
  `level_1.tscn`, `level_2.tscn`, `level_3.tscn`, etc.
- The script relies on extracting the level number from the scene name.

### Node Structure

````text
LevelChanger (Area2D)
└── CollisionShape2D

## Script

```gdscript
extends Area2D

"""
LevelChanger.gd
----------------

This script handles transitioning the player to the next level when
the player enters the Area2D.

Attach this script to an `Area2D` node with a collision shape.
Connect the `body_entered` signal to `_on_body_entered`.

Scene naming convention:
    level_1.tscn, level_2.tscn, level_3.tscn, etc.
"""

# ============================================================
# SIGNAL CALLBACKS
# ============================================================

func _on_body_entered(body: Node2D) -> void:
    """
    Triggered when a body enters the Area2D.

    Parameters:
        body (Node2D): The body that entered the area.

    Behavior:
        1. Determines the current level number from the scene name.
        2. Loads the next level scene by incrementing the current level number.
    """

    # Optional: only react if the entering body is the player
    if not body.is_in_group("Player"):
        return

    # Determine current level number from scene name
    var scene_name := get_tree().current_scene.name
    var level_number := scene_name.get_slice("_", 1).to_int()  # expects name like "level_1"

    # Construct the next level scene path
    var next_level_path := "res://level_" + str(level_number + 1) + ".tscn"

    # Safely change to the next level if the file exists
    if FileAccess.file_exists(next_level_path):
        get_tree().change_scene_to_file(next_level_path)
    else:
        push_warning("Next level scene not found: " + next_level_path)


````
