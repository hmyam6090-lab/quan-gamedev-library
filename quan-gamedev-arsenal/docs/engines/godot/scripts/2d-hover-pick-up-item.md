# Hovering/Pickupable 2D Item (Godot 4.5)

Quan Hoang's reusable 2D item script for Godot 4.5.

---

## Module Overview

**Root Node:** `Node2D`  
**Engine:** Godot 4.5

This script makes a 2D item hover smoothly and allows the player (or other bodies) to pick it up when they touch it. The item automatically removes itself from the scene upon pickup and optionally plays a sound effect.

---

## Features

- Smooth vertical hovering using sine wave motion
- Configurable hover amplitude and speed
- Detects body collisions via `Area2D` and triggers pickup
- Optional pickup sound effect
- Automatically removes itself from the scene when picked up
- Easy to extend for different types of items, power-ups, or inventory objects

---

## Prerequisites

### Scene Requirements

The item scene should follow this structure:

Item (Node2D) ← Attach this script

├── Area2D

│ └── CollisionShape2D

└── AudioStreamPlayer2D ← Optional, assign to `pickup_sound`

- The root node must be `Node2D`
- An `Area2D` with a `CollisionShape2D` is required to detect pickups
- The `AudioStreamPlayer2D` is optional for sound effects

---

### Input / Signals

- Pickup is triggered automatically when a `Node2D` enters the `Area2D`
- You can optionally call a function or signal in the player to give the item
- Example: `body.obtain_item(self)` inside `_on_area_2d_body_entered`

---

## Exported Properties

| Property          | Description                                   | Default |
| ----------------- | --------------------------------------------- | ------- |
| `hover_amplitude` | Maximum vertical movement from the start Y    | 6.0     |
| `hover_speed`     | Speed of vertical sine-wave oscillation       | 3.0     |
| `pickup_sound`    | Optional AudioStreamPlayer2D node to play SFX | null    |

---

## Script Overview

```gdscript
extends Node2D

@export var hover_amplitude: float = 6.0
@export var hover_speed: float = 3.0
@export var pickup_sound: AudioStreamPlayer2D

var _time: float = 0.0
var _start_y: float = 0.0

func _ready() -> void:
	_start_y = position.y

func _physics_process(delta: float) -> void:
	_time += delta
	position.y = _start_y + sin(_time * hover_speed) * hover_amplitude

func _on_area_2d_body_entered(body: Node2D) -> void:
	if pickup_sound:
		pickup_sound.play()
	queue_free()
```
