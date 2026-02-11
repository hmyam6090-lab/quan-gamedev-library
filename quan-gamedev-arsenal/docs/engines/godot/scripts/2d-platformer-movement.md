# 2D Platformer Player Controller (Godot 4.5)

**Author:** Quan Hoang  
**Engine:** Godot 4.5  
**Root Node:** `CharacterBody2D`

A modular, reusable 2D platformer player controller with advanced movement mechanics, including jumps with coyote time and jump buffering, wall jumps, dash, particles, audio, animations, and optional UI feedback.

---

## Features

- **Horizontal Movement** with acceleration and friction
- **Jumping Mechanics**:
  - **Coyote Time** – jump shortly after leaving the ground
  - **Jump Buffer** – registers early jump input
  - **Double Jump** – optional
- **Wall Interactions**:
  - Wall slide
  - Wall jump with directional push
- **Dash Mechanic** with afterimages and cooldown
- **Gravity & Terminal Velocity** control
- `AnimatedSprite2D` animation control with automatic sprite flipping
- **Sound Effects (optional)**: jump, land, footsteps, dash, death
- **Particle Effects (optional)**: jump dust, landing dust, run dust, wall slide
- **Camera Shake** for jump/dash feedback
- **UI Feedback (optional)**: speed and state labels

---

## Prerequisites

### Scene Requirements

Attach this script to the **root node of your player scene**, which must be a `CharacterBody2D`. The recommended node hierarchy is:

Player (CharacterBody2D) ← Attach this script
├── AnimatedSprite2D

├── CollisionShape2D

├── JumpSFX (AudioStreamPlayer2D) [optional]

├── LandSFX (AudioStreamPlayer2D) [optional]

├── FootstepSFX (AudioStreamPlayer2D) [optional]

├── JumpParticles (GPUParticles2D) [optional]

├── LandParticles (GPUParticles2D) [optional]

├── RunParticles (GPUParticles2D) [optional]

**Optional UI:**

UI (CanvasLayer) ← Optional

├── SpeedLabel (Label)

└── StateLabel (Label)

- The root node of the player must be a `CharacterBody2D`
- The animation node must be named exactly `AnimatedSprite2D`
- Sound effects, particles, and UI elements are optional

**Notes:**

- The script automatically checks for optional nodes, so missing nodes will not break functionality.
- The `AnimatedSprite2D` node **must** be present for animations.
- Audio, particles, and UI elements are optional.

---

## Required Animations

Your `AnimatedSprite2D` should define the following animations:

| Player State | Animation Name                       |
| ------------ | ------------------------------------ |
| Idle         | `idle`                               |
| Walk         | `walk`                               |
| Jump         | `jump`                               |
| Fall         | `fall`                               |
| Dash         | `dash` (optional if dash enabled)    |
| Die          | `die` (optional if death is handled) |

---

> Animation names are **case-sensitive**.

---

## Input Map

Define these input actions in **Project Settings → Input Map**:

| Action  | Default Keys            |
| ------- | ----------------------- |
| `left`  | A, Left Arrow           |
| `right` | D, Right Arrow          |
| `jump`  | Space, W, Up Arrow      |
| `dash`  | Shift / optional        |
| `r`     | Reload scene (optional) |

---

> All other inputs (like dash, reload) are optional depending on your game design.

---

## Optional Features

- **UI Feedback**:
  - `SpeedLabel` → displays current horizontal speed
  - `StateLabel` → displays player state (`Idle`, `Running`, `Airborne`)

- **Particles**:
  - `JumpParticles` → plays on jump
  - `LandParticles` → plays on landing
  - `RunParticles` → emits while running
  - `WallSlideParticles` → emits when sliding on wall
  - Custom smoke/afterimage effects supported

- **Sound Effects**:
  - Jump, land, dash, footsteps, switch, death

- **Camera Shake**: reacts to jump and dash

- **Dash Mechanic**: optional; configurable speed, cooldown, and afterimage effect

---

## Notes on Customization

- All exported variables are organized with `@export_group` for easy tweaking in the Inspector.
- Mechanics such as **double jump**, **dash**, and **wall interactions** are fully configurable.
- All optional nodes and effects are safely ignored if not assigned.
- The controller is modular and designed to be reused across multiple levels or projects.

## Script

```gdscript
## ============================================================
##  QUAN HOANG 2D PLATFORMER PLAYER CONTROLLER (GODOT 4.5)
## ============================================================
##
##  Fully-featured 2D platformer player controller.
##  Features:
##  - Horizontal movement with acceleration & friction
##  - Jumping (Coyote Time, Jump Buffer, Double Jump)
##  - Wall jumps & wall slides
##  - Dash with afterimages
##  - Gravity & terminal velocity
##  - AnimatedSprite2D animation control
##  - Optional audio & particle effects
##  - Optional UI feedback
##  - Camera shake for jump/dash effects
##  - Clean, modular structure with @export_group
##
## ============================================================

extends CharacterBody2D

# ============================================================
# MOVEMENT SETTINGS
# ============================================================
@export_group("Movement Settings")
@export var max_speed: float = 170.0
@export var acceleration: float = 1200.0
@export var friction: float = 1400.0

# ============================================================
# JUMP SETTINGS
# ============================================================
@export_group("Jump Settings")
@export var jump_velocity: float = -300.0
@export var gravity: float = 1100.0
@export var terminal_velocity: float = 1200.0
@export var coyote_time: float = 0.12
@export var jump_buffer_time: float = 0.15
@export var can_double_jump: bool = true
var _max_jumps = 1
var _jump_count = 0

# Wall jump / slide
@export_group("Wall Jump Settings")
@export var gravity_wall: float = 6.5
@export var max_wall_slide_speed: float = 60
@export var wall_jump_push_force: float = 250
@export var wall_contact_coyote_time: float = 0.15
@export var wall_jump_lock_time: float = 0.5
@export var wall_jump_input_boost: float = 30

var wall_contact_coyote: float = 0.0
var wall_jump_lock: float = 0.0
var look_dir_x: int = 1

# ============================================================
# DASH SETTINGS
# ============================================================
@export_group("Dash Settings")
@export var dash_force: float = 500.0
@export var dash_buffer_time: float = 0.15
@export var dash_cooldown: float = 3.0
@export var afterimage_drag_time: float = 0.15
@export var can_dash: bool = false

var _is_dashing: bool = false
var _dash_available: bool = true
var _cancel_gravity: bool = false
var _dash_buffer_timer: float = 0.0
var _afterimage = preload("res://Scenes/afterimage.tscn")

# ============================================================
# ANIMATION REFERENCES
# ============================================================
@export_group("Animation")
@export var animated_sprite: AnimatedSprite2D

# ============================================================
# AUDIO REFERENCES (OPTIONAL)
# ============================================================
@export_group("Sound Effects")
@export var dash_sfx: AudioStreamPlayer2D
@export var jump_sfx: AudioStreamPlayer2D
@export var land_sfx: AudioStreamPlayer2D
@export var footstep_sfx: AudioStreamPlayer2D
@export var switch_sfx: AudioStreamPlayer2D
@export var death_sfx: AudioStreamPlayer2D
@export var flashlight_sfx: AudioStreamPlayer2D
@export var walktimer: Timer

@export_group("Music")
@export var main_track: AudioStreamPlayer2D

# ============================================================
# PARTICLES (OPTIONAL)
# ============================================================
@export_group("Particles")
@export var dash_particles: GPUParticles2D
@export var jump_particles: GPUParticles2D
@export var land_particles: GPUParticles2D
@export var run_particles: GPUParticles2D
@export var wall_slide_particles_right: GPUParticles2D
@export var wall_slide_particles_left: GPUParticles2D
var _smoke = preload("res://Scenes/smoke.tscn")

# ============================================================
# CAMERA / JUICE
# ============================================================
@export_group("Game Juice")
@export var camera: Camera2D
@export var jump_shake_strength: float = 1.0
@export var jump_shake_duration: float = 0.08
var shake_timer: float = 0.0
var shake_strength: float = 0.0
var camera_offset: Vector2 = Vector2.ZERO
var shake_decay: float = 5.0
var shake_time_speed: float = 2.0

# ============================================================
# UI (OPTIONAL)
# ============================================================
@export_group("UI")
@export var speed_label: Label
@export var state_label: Label

# ============================================================
# PLAYER STATE (INTERNAL)
# ============================================================
@export_group("Player State")
var _is_dead: bool = false

var _input_dir: float = 0.0
var _coyote_timer: float = 0.0
var _jump_buffer_timer: float = 0.0
var _was_on_floor: bool = false
var _is_running: bool = false
var _footsteps_played: bool = false
@export var _is_reload: bool = true

# ============================================================
# GODOT LIFECYCLE
# ============================================================
func _physics_process(delta: float) -> void:
	if _is_dead:
		return

	if $"../ReloadScreen" and _is_reload:
		return

	read_input()
	apply_gravity(delta)
	handle_horizontal_movement(delta)
	move_and_slide()
	handle_jump_and_wall_jump(delta)
	handle_dash(delta)
	handle_landing()
	update_animation()
	update_particles()
	update_ui()
	handle_camera_shake(delta)

	if _is_dashing:
		instance_afterimage()

# ============================================================
# INPUT HANDLING
# ============================================================
func read_input() -> void:
	"""
	Reads player input for horizontal movement, jumping, dash, reload, and world switching.
	"""
	_input_dir = Input.get_action_strength("right") - Input.get_action_strength("left")

	# Jump buffer
	if Input.is_action_just_pressed("jump"):
		_jump_buffer_timer = jump_buffer_time

	# Dash input
	if can_dash and Input.is_action_just_pressed("dash") and _dash_available:
		start_dash()

	# Reload scene
	if Input.is_action_pressed("r"):
		get_tree().reload_current_scene()

# ============================================================
# MOVEMENT
# ============================================================
func handle_horizontal_movement(delta: float) -> void:
	if _input_dir != 0:
		velocity.x = move_toward(velocity.x, _input_dir * max_speed, acceleration * delta)
	else:
		velocity.x = move_toward(velocity.x, 0, friction * delta)

func apply_gravity(delta: float) -> void:
	if not is_on_floor():
		if not _cancel_gravity:
			velocity.y += gravity * delta
			velocity.y = min(velocity.y, terminal_velocity)
	else:
		_dash_available = true
		_jump_count = 0
		_coyote_timer = coyote_time

# ============================================================
# JUMP / WALL JUMP
# ============================================================
func handle_jump_and_wall_jump(delta: float) -> void:
	_coyote_timer -= delta
	_jump_buffer_timer -= delta
	wall_contact_coyote = max(wall_contact_coyote - delta, 0.0)

	# Horizontal lock after wall jump
	if wall_jump_lock > 0.0:
		wall_jump_lock -= delta
	else:
		velocity.x = lerp(velocity.x, _input_dir * max_speed, acceleration * 0.0001)

	# Wall slide
	var on_wall := !is_on_floor() and is_on_wall()
	if on_wall and velocity.y > 0:
		handle_wall_slide()

	# Jump logic
	if Input.is_action_just_pressed("jump"):
		if wall_contact_coyote > 0.0 and not is_on_floor():
			do_wall_jump()
			return
		elif _coyote_timer > 0.0:
			do_jump()

func handle_wall_slide() -> void:
	"""
	Handles wall slide particles and gravity.
	"""
	if not animated_sprite.flip_h:
		wall_slide_particles_right.emitting = true
	else:
		wall_slide_particles_left.emitting = true

	var wall_dir = -get_wall_normal().x
	look_dir_x = wall_dir
	wall_contact_coyote = wall_contact_coyote_time

	velocity.y += gravity_wall * get_process_delta_time()
	velocity.y = min(velocity.y, max_wall_slide_speed)

func do_jump() -> void:
	instance_smoke()
	velocity.y = jump_velocity
	_jump_buffer_timer = 0.0
	_coyote_timer = 0.0
	play_jump_effects()

func do_wall_jump() -> void:
	var wall_dir = -look_dir_x
	velocity.y = jump_velocity
	velocity.x = wall_dir * wall_jump_push_force + _input_dir * wall_jump_input_boost
	wall_jump_lock = wall_jump_lock_time
	wall_contact_coyote = 0.0
	_jump_buffer_timer = 0.0
	_coyote_timer = 0.0
	play_jump_effects()

# ============================================================
# DASH
# ============================================================
func handle_dash(delta: float) -> void:
	_dash_buffer_timer -= delta
	if _dash_buffer_timer > 0.0:
		velocity.y = 0
		velocity.x = _input_dir * dash_force
		_dash_buffer_timer = 0
		play_dash_effects()

func start_dash() -> void:
	_dash_available = false
	_is_dashing = true
	_cancel_gravity = true
	_dash_buffer_timer = dash_buffer_time
	await get_tree().create_timer(0.3).timeout
	_cancel_gravity = false
	_is_dashing = false

# ============================================================
# LANDING
# ============================================================
func handle_landing() -> void:
	if not _was_on_floor and is_on_floor():
		play_land_effects()
	_was_on_floor = is_on_floor()

# ============================================================
# VISUALS & PARTICLES
# ============================================================
func update_animation() -> void:
	if not animated_sprite:
		return
	if _is_dead:
		animated_sprite.speed_scale = 2
		animated_sprite.play("die")
		return

	if not is_on_floor():
		animated_sprite.play("jump" if velocity.y < 0 else "fall")
	elif abs(velocity.x) > 10:
		if _is_dashing:
			animated_sprite.play("dash")
		else:
			animated_sprite.speed_scale = 2
			animated_sprite.play("walk")
	else:
		animated_sprite.speed_scale = 1
		animated_sprite.play("idle")

	if _input_dir != 0:
		animated_sprite.flip_h = _input_dir < 0

func update_particles() -> void:
	if run_particles:
		run_particles.emitting = is_on_floor() and abs(velocity.x) > 20
		run_particles.process_material.gravity = Vector3(_input_dir * 0.1, -2, 0)
		run_particles.position.x = _input_dir * 5

# ============================================================
# EFFECTS
# ============================================================
func play_jump_effects() -> void:
	if jump_sfx:
		jump_sfx.play()
	if jump_particles:
		jump_particles.restart()

func play_dash_effects() -> void:
	shake_timer = jump_shake_duration
	shake_strength = jump_shake_strength * 3
	if dash_sfx:
		dash_sfx.play()
	if dash_particles:
		dash_particles.restart()

func play_land_effects() -> void:
	if land_sfx:
		land_sfx.play()
	if land_particles:
		land_particles.restart()

# ============================================================
# AFTERIMAGES & SMOKE
# ============================================================
func instance_afterimage() -> void:
	var afterimage_instance: Sprite2D = _afterimage.instantiate()
	get_parent().get_parent().add_child(afterimage_instance)
	afterimage_instance.global_position = self.global_position
	afterimage_instance.flip_h = _input_dir < 0
	afterimage_instance.z_index = self.z_index + 1

	var tween = get_tree().create_tween()
	tween.tween_property(afterimage_instance, "modulate:a", 0, afterimage_drag_time).set_trans(Tween.TRANS_SINE)
	tween.tween_callback(afterimage_instance.queue_free)

func instance_smoke() -> void:
	var smoke_instance: AnimatedSprite2D = _smoke.instantiate()
	get_parent().get_parent().add_child(smoke_instance)
	smoke_instance.global_position = self.global_position + Vector2(0, 5)
	await get_tree().create_timer(0.4).timeout
	smoke_instance.queue_free()

# ============================================================
# CAMERA
# ============================================================
func handle_camera_shake(delta: float) -> void:
	if shake_timer > 0.0:
		shake_timer -= delta
		var t := shake_timer / jump_shake_duration
		var intensity := shake_strength * t
		camera_offset = Vector2(randf_range(-1, 1), randf_range(-1, 1)) * intensity
	else:
		camera_offset = camera_offset.lerp(Vector2.ZERO, 15 * delta)
	camera.offset = camera_offset

# ============================================================
# UI
# ============================================================
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

# ============================================================
# PLAYER DEATH
# ============================================================
func die() -> void:
	if _is_dead:
		return
	_is_dead = true
	if death_sfx:
		death_sfx.play()

	shake_timer = jump_shake_duration
	shake_strength = jump_shake_strength * 15
	velocity = Vector2(1, 1)
	self.motion_mode = CharacterBody2D.MOTION_MODE_FLOATING
	_cancel_gravity = true
	await get_tree().create_timer(1).timeout
	queue_free()
	get_tree().reload_current_scene()

```
