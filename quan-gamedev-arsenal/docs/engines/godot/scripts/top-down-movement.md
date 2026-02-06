# Top-Down Player Controller (Godot 4.5)

Quan Hoang's reusable top-down 2D player controller for Godot 4.5.

---

## Module Overview

**Root Node:** `CharacterBody2D`  
**Engine:** Godot 4.5

---

## Features

- 8-directional movement using normalized input
- Optional sprinting
- Optional stamina drain and regeneration
- `AnimatedSprite2D` animation control
- Optional UI feedback (speed label and stamina bar)

---

## Prerequisites

### Scene Requirements

The player scene must follow this structure:

Player (CharacterBody2D)

├── AnimatedSprite2D

├── CollisionShape2D

└── GPUParticles2D ← Optional

UI (CanvasLayer) ← Optional

├── SpeedLabel (Label)

└── StaminaBar (ProgressBar)

- The root node must be `CharacterBody2D`
- The root node is expected to be named `Player`
- The animation node must be named exactly `AnimatedSprite2D`

---

### Required Animations

The assigned `AnimatedSprite2D` must define the following animations:

**Idle Animations**

- `idle_up`
- `idle_down`
- `idle_left`
- `idle_right`

**Walking Animations**

- `walk_up`
- `walk_down`
- `walk_left`
- `walk_right`

Animation names are case-sensitive.

---

### Input Map

The following input actions must be defined in **Project Settings → Input Map**:

| Action   | Expected Keys  |
| -------- | -------------- |
| `left`   | A, Left Arrow  |
| `right`  | D, Right Arrow |
| `up`     | W, Up Arrow    |
| `down`   | S, Down Arrow  |
| `sprint` | Shift          |

---

### Optional UI Elements

If UI feedback is desired, create a `CanvasLayer` containing:

- `Label` for displaying movement speed
- `ProgressBar` for displaying stamina

Both references are optional and safely ignored if not assigned.

---

## Exported Properties

### Movement Settings

```gdscript
@export_range(0, 1000)
var walk_speed: float = 120.0
```

---

## Script

```gdscript
extends CharacterBody2D

# ============================================================
# MOVEMENT SETTINGS
# ============================================================
@export_group("Movement Settings")

@export_range(0, 1000)
var walk_speed: float = 120.0

@export_range(0, 1500)
var sprint_speed: float = 200.0

@export
var allow_sprinting: bool = true


# ============================================================
# STAMINA SETTINGS (OPTIONAL)
# ============================================================
@export_group("Stamina Settings")

@export
var use_stamina: bool = true

@export_range(0, 100)
var max_stamina: float = 100.0

@export_range(0, 100)
var stamina_drain_per_second: float = 25.0

@export_range(0, 100)
var stamina_recovery_per_second: float = 15.0


# ============================================================
# ANIMATION REFERENCES
# ============================================================
@export_group("Animation")

@export
var animated_sprite: AnimatedSprite2D


# ============================================================
# UI REFERENCES (OPTIONAL)
# ============================================================
@export_group("UI")

@export
var speed_label: Label

@export
var stamina_bar: ProgressBar


# ============================================================
# INTERNAL STATE (NOT EXPORTED)
# ============================================================
var _input_direction: Vector2 = Vector2.ZERO
var _current_speed: float
var _current_stamina: float


# ============================================================
# GODOT LIFECYCLE
# ============================================================
func _ready() -> void:
	_current_stamina = max_stamina
	update_ui()


func _physics_process(delta: float) -> void:
	read_player_input()
	apply_movement(delta)
	move_and_slide()
	update_animation()
	update_ui()


# ============================================================
# INPUT & MOVEMENT
# ============================================================

## read_player_input() reads directional input from the Input Map
## and stores it as a normalized Vector2.
func read_player_input() -> void:
	_input_direction = Input.get_vector("left", "right", "up", "down")


## apply_movement() applies velocity based on input, sprinting,
## and stamina rules.
func apply_movement(delta: float) -> void:
	var is_sprinting := Input.is_action_pressed("sprint") and allow_sprinting

	if is_sprinting and can_sprint():
		_current_speed = sprint_speed
		drain_stamina(delta)
	else:
		_current_speed = walk_speed
		recover_stamina(delta)

	velocity = _input_direction * _current_speed


## can_sprint() returns true if the player is allowed to sprint.
func can_sprint() -> bool:
	if not use_stamina:
		return true
	return _current_stamina > 0


# ============================================================
# STAMINA HANDLING
# ============================================================

## Reduces stamina while sprinting.
func drain_stamina(delta: float) -> void:
	if not use_stamina:
		return

	_current_stamina -= stamina_drain_per_second * delta
	_current_stamina = max(_current_stamina, 0)


## Regenerates stamina when not sprinting.
func recover_stamina(delta: float) -> void:
	if not use_stamina:
		return

	_current_stamina += stamina_recovery_per_second * delta
	_current_stamina = min(_current_stamina, max_stamina)


# ============================================================
# ANIMATION CONTROL
# ============================================================

## Updates AnimatedSprite2D based on movement direction.
## Expects animations (name it EXACTLY this):
##   - "idle_up", "idle_down", "idle_left", "idle_right"
##   - "walk_up", "walk_down", "walk_left", "walk_right"
func update_animation() -> void:
	if animated_sprite == null:
		return

	if _input_direction == Vector2.ZERO:
		animated_sprite.play("idle_down")
		return

	if abs(_input_direction.x) > abs(_input_direction.y):
		animated_sprite.play("walk_right" if _input_direction.x > 0 else "walk_left")
	else:
		animated_sprite.play("walk_down" if _input_direction.y > 0 else "walk_up")


# ============================================================
# UI UPDATES
# ============================================================

## Updates UI elements if they are assigned.
func update_ui() -> void:
	if speed_label:
		speed_label.text = "Speed: " + str(int(_current_speed))

	if stamina_bar:
		stamina_bar.max_value = max_stamina
		stamina_bar.value = _current_stamina
```
