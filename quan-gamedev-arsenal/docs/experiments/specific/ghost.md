# Ghost Enemy Controller (Godot 4.5)

Quan Hoang's reusable ghost enemy controller for Godot 4.5.

---

## Module Overview

**Root Node:** `CharacterBody2D`  
**Engine:** Godot 4.5

This script controls a ghost enemy that continuously chases the player and kills them if they get too close. It supports animation flipping, optional sound effects, and smooth movement.

---

## Features

- Chases the player using normalized movement
- Flips sprite based on movement direction
- Optional sound effect when the ghost becomes active
- Kills the player on contact (configurable proximity)
- Simple, reusable, and safe structure

---

## Prerequisites

### Scene Requirements

The ghost enemy scene must follow this structure:

GhostEnemy (CharacterBody2D) ← Attach this script

├── AnimatedSprite2D ← Required for animations and flipping

├── CollisionShape2D ← Optional, required only if using physics interactions

└── GhostSFX (AudioStreamPlayer2D) ← Optional, plays when ghost activates

- The root node of the ghost must be a `CharacterBody2D`
- The animation node must be named exactly `AnimatedSprite2D`
- Sound effect node is optional

---

### Input Map

No input map is required — the ghost is fully AI-controlled.

---

### Optional Parameters

- `speed` — movement speed of the ghost (default: 20)
- `anim_sprite` — reference to the ghost’s AnimatedSprite2D
- `coll` — reference to the CollisionShape2D (optional)
- `ghost_sfx` — optional sound effect played once when ghost activates
- `ghost_sfx_played` — internal flag to prevent repeated SFX

---

## Behavior

- The ghost moves continuously toward the player every frame.
- The `AnimatedSprite2D` flips horizontally based on the ghost's movement direction.
- If the ghost is within **10 units** of the player, it calls `player.die()`.
- The optional sound effect plays once when the ghost first becomes active.

---

## Usage Notes

- Attach this script directly to your ghost enemy node.
- Assign the `AnimatedSprite2D` and optional `AudioStreamPlayer2D` nodes in the inspector.
- Ensure there is a `Player` node in the same parent scene so the ghost can find it.
- Adjust `speed` and kill distance as needed for gameplay balance.

```gdscript

extends CharacterBody2D

# ============================================================
#  GHOST ENEMY CONTROLLER (GODOT 4.5)
# ============================================================
#
# This script controls a ghost enemy that:
# - Moves toward the player
# - Plays animations and optional sound effects
# - Kills the player if it gets too close
#
# Attach this to the ghost enemy node (CharacterBody2D).
# ============================================================

# ============================================================
# EXPORT VARIABLES
# ============================================================
@export var speed: float = 20
@export var anim_sprite: AnimatedSprite2D
@export var coll: CollisionShape2D  # optional if using collisions
@export var ghost_sfx: AudioStreamPlayer2D

# ============================================================
# INTERNAL STATE
# ============================================================
var ghost_sfx_played: bool = false

# ============================================================
# GODOT LIFECYCLE
# ============================================================

func _ready() -> void:
	# Ghost starts visible
	set_visible(true)

func _process(delta: float) -> void:
	handle_sfx()
	chase_player(delta)

# ============================================================
# HELPER FUNCTIONS
# ============================================================

# Plays the ghost sound effect once
func handle_sfx() -> void:
	if not ghost_sfx_played and ghost_sfx:
		ghost_sfx.play()
		ghost_sfx_played = true

# Moves the ghost toward the player and kills the player on contact
func chase_player(delta: float) -> void:
	var player = $"../Player"
	if player == null:
		return

	# Move toward the player
	var direction = (player.global_position - global_position).normalized()
	velocity = direction * speed
	move_and_slide()

	# Flip sprite based on horizontal movement
	if anim_sprite:
		anim_sprite.flip_h = velocity.x < 0

	# Kill player if too close
	var distance = global_position.distance_to(player.global_position)
	if distance < 10:
		player.die()


```
