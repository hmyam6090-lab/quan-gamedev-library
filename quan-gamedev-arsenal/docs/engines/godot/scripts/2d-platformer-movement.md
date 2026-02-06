# 2D Platformer Player Controller (Godot 4.5)

Quan Hoang's reusable 2D platformer player controller for Godot 4.5.

---

## Module Overview

**Root Node:** `CharacterBody2D`  
**Engine:** Godot 4.5

This script provides full 2D platformer functionality including movement, jumping with coyote time and jump buffering, gravity handling, animation, particles, sound effects, and optional UI feedback.

---

## Features

- Horizontal movement with acceleration and friction
- Jumping with:
  - **Coyote Time** (allows jumping shortly after leaving the ground)
  - **Jump Buffer** (registers early jump input)
- Gravity and terminal velocity control
- `AnimatedSprite2D` animation control with sprite flipping
- Optional sound effects (jump, land, footsteps)
- Optional particle effects (jump dust, landing dust, run dust)
- Optional UI labels for speed and state

---

## Prerequisites

### Scene Requirements

The player scene must follow this structure:

Player (CharacterBody2D) ← Attach this script

├── AnimatedSprite2D

├── CollisionShape2D

├── JumpSFX (AudioStreamPlayer2D) [optional]

├── LandSFX (AudioStreamPlayer2D) [optional]

├── FootstepSFX (AudioStreamPlayer2D) [optional]

├── JumpParticles (GPUParticles2D) [optional]

├── LandParticles (GPUParticles2D) [optional]

└── RunParticles (GPUParticles2D) [optional]

UI (CanvasLayer) ← Optional

├── SpeedLabel (Label)

└── StateLabel (Label)

- The root node of the player must be a `CharacterBody2D`
- The animation node must be named exactly `AnimatedSprite2D`
- Sound effects, particles, and UI elements are optional

---

### Required Animations

The assigned `AnimatedSprite2D` must define the following animations:

| State | Animation Name |
| ----- | -------------- |
| Idle  | `idle`         |
| Walk  | `walk`         |
| Jump  | `jump`         |
| Fall  | `fall`         |

Animation names are **case-sensitive**.

---

### Input Map

The following input actions must be defined in **Project Settings → Input Map**:

| Action  | Expected Keys      |
| ------- | ------------------ |
| `left`  | A, Left Arrow      |
| `right` | D, Right Arrow     |
| `jump`  | Space, W, Up Arrow |

---

### Optional UI Elements

If UI feedback is desired, create a `CanvasLayer` containing:

- `SpeedLabel` (Label) to display horizontal speed
- `StateLabel` (Label) to display current player state (Idle, Running, Airborne)

Both references are optional and safely ignored if not assigned.

---

## Script

```gdscript
## ============================================================
##  QUAN HOANG 2D PLATFORMER PLAYER CONTROLLER (GODOT 4.5)
## ============================================================
##
##  This script implements a 2D platformer player controller.
##
##  ------------------------------------------------------------
##  FEATURES
##  ------------------------------------------------------------
##  - Horizontal movement with acceleration & friction
##  - Jumping with:
##      • Coyote Time (jump after leaving ground)
##      • Jump Buffer (press jump slightly early)
##  - Gravity & terminal velocity
##  - AnimatedSprite2D animation control
##  - Sprite flipping
##  - Sound effects (jump / land / footsteps)
##  - Particle effects (jump dust / landing dust / run dust)
##  - Optional UI feedback (speed label, state label)
##
##  ------------------------------------------------------------
##  PREREQUISITES — NODE TREE
##  ------------------------------------------------------------
##
##  Main (Node2D)
##  ├── Player (CharacterBody2D)   ← Attach this script
##  │   ├── AnimatedSprite2D
##  │   ├── CollisionShape2D
##  │   ├── JumpSFX (AudioStreamPlayer2D)        [optional]
##  │   ├── LandSFX (AudioStreamPlayer2D)        [optional]
##  │   ├── FootstepSFX (AudioStreamPlayer2D)   [optional]
##  │   ├── JumpParticles (GPUParticles2D)      [optional]
##  │   ├── LandParticles (GPUParticles2D)      [optional]
##  │   └── RunParticles (GPUParticles2D)       [optional]
##  │
##  └── UI (CanvasLayer)                         [optional]
##      ├── SpeedLabel (Label)
##      └── StateLabel (Label)
##
##  ------------------------------------------------------------
##  REQUIRED INPUT MAP
##  ------------------------------------------------------------
##   - "left"   → A / Left Arrow
##   - "right"  → D / Right Arrow
##   - "jump"   → Space / W / Up Arrow
##
##  ------------------------------------------------------------
##  ANIMATION NAMES (AnimatedSprite2D)
##  ------------------------------------------------------------
##   - "idle"
##   - "walk"
##   - "jump"
##   - "fall"
##
## ============================================================

extends CharacterBody2D

# ============================================================
# MOVEMENT SETTINGS
# ============================================================
@export_group("Movement Settings")

@export var max_speed: float = 220.0
@export var acceleration: float = 1200.0
@export var friction: float = 1400.0


# ============================================================
# JUMP SETTINGS
# ============================================================
@export_group("Jump Settings")

@export var jump_velocity: float = -380.0
@export var gravity: float = 1100.0
@export var terminal_velocity: float = 1200.0

@export var coyote_time: float = 0.12
@export var jump_buffer_time: float = 0.15

# ============================================================
# ANIMATION
# ============================================================
@export_group("Animation")

@export var animated_sprite: AnimatedSprite2D


# ============================================================
# AUDIO (OPTIONAL)
# ============================================================
@export_group("Sound Effects")

@export var jump_sfx: AudioStreamPlayer2D
@export var land_sfx: AudioStreamPlayer2D
@export var footstep_sfx: AudioStreamPlayer2D


# ============================================================
# PARTICLES (OPTIONAL)
# ============================================================
@export_group("Particles")

@export var jump_particles: GPUParticles2D
@export var land_particles: GPUParticles2D
@export var run_particles: GPUParticles2D


# ============================================================
# UI (OPTIONAL)
# ============================================================
@export_group("UI")

@export var speed_label: Label
@export var state_label: Label


# ============================================================
# INTERNAL STATE
# ============================================================
var _input_dir: float = 0.0
var _coyote_timer: float = 0.0
var _jump_buffer_timer: float = 0.0
var _was_on_floor: bool = false


# ============================================================
# GODOT LIFECYCLE
# ============================================================
func _physics_process(delta: float) -> void:
	read_input()
	apply_gravity(delta)
	handle_jump(delta)
	handle_horizontal_movement(delta)
	move_and_slide()
	handle_landing()
	update_animation()
	update_particles()
	update_ui()


# ============================================================
# INPUT
# ============================================================

## Reads left/right input.
func read_input() -> void:
	_input_dir = Input.get_action_strength("right") - Input.get_action_strength("left")

	if Input.is_action_just_pressed("jump"):
		_jump_buffer_timer = jump_buffer_time


# ============================================================
# PHYSICS
# ============================================================

## Applies gravity and clamps fall speed.
func apply_gravity(delta: float) -> void:
	if not is_on_floor():
		velocity.y += gravity * delta
		velocity.y = min(velocity.y, terminal_velocity)
	else:
		_coyote_timer = coyote_time


## Handles jump logic (coyote time + buffer).
func handle_jump(delta: float) -> void:
	_coyote_timer -= delta
	_jump_buffer_timer -= delta

	if _jump_buffer_timer > 0 and _coyote_timer > 0:
		velocity.y = jump_velocity
		_jump_buffer_timer = 0
		_coyote_timer = 0
		play_jump_effects()


## Handles acceleration & friction.
func handle_horizontal_movement(delta: float) -> void:
	if _input_dir != 0:
		velocity.x = move_toward(
			velocity.x,
			_input_dir * max_speed,
			acceleration * delta
		)
	else:
		velocity.x = move_toward(
			velocity.x,
			0,
			friction * delta
		)


## Detects landing events.
func handle_landing() -> void:
	if not _was_on_floor and is_on_floor():
		play_land_effects()
	_was_on_floor = is_on_floor()


# ============================================================
# VISUALS
# ============================================================

## Updates animation and sprite direction.
func update_animation() -> void:
	if not animated_sprite:
		return

	if not is_on_floor():
		animated_sprite.play("jump" if velocity.y < 0 else "fall")
	elif abs(velocity.x) > 10:
		animated_sprite.play("walk")
	else:
		animated_sprite.play("idle")

	if _input_dir != 0:
		animated_sprite.flip_h = _input_dir < 0


## Controls particle emission.
func update_particles() -> void:
	if run_particles:
		run_particles.emitting = is_on_floor() and abs(velocity.x) > 20


# ============================================================
# EFFECTS
# ============================================================

## Plays jump sound & particles.
func play_jump_effects() -> void:
	if jump_sfx:
		jump_sfx.play()
	if jump_particles:
		jump_particles.restart()


## Plays landing sound & particles.
func play_land_effects() -> void:
	if land_sfx:
		land_sfx.play()
	if land_particles:
		land_particles.restart()


# ============================================================
# UI
# ============================================================

## Updates UI labels if assigned.
func update_ui() -> void:
	if speed_label:
		speed_label.text = "Speed: " + str(int(abs(velocity.x)))

	if state_label:
		if not is_on_floor():
			state_label.text = "Airborne"
		elif abs(velocity.x) > 10:
			state_label.text = "Running"
		else:
			state_label.text = "Idle"

```
