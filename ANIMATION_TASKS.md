# Character Animation Build-To-Dos List

**Project:** Rooftop Peekaboo  
**Last Updated:** 2026-10-27  
**Status:** Planning Phase

---

## Overview

Parkour-style character movement animation system with wall-running, corner-hugging, and dynamic crouching mechanics. Focus on fluid transitions between states for responsive gameplay.

---

## Animation States & Transitions

### 1. 🧱 Hug Corner
**Description:** Character slides along a corner edge while maintaining contact with both walls.

- [ ] **Animation Clip:** `anim_hug_corner` (looping)
  - Body angled into corner
  - Arms braced against walls
  - Legs in sliding motion
- [ ] **Blend Space:** Corner angle variation (0°–90°)
- [ ] **Transition In:** From wall-run or air state when corner detected
- [ ] **Transition Out:** To wall-run, jump, or ground state
- [ ] **Input Mapping:** Automatic on corner collision during wall movement
- [ ] **Physics:** Reduced gravity, friction-based slide speed

### 2. 🦵 Crouch
**Description:** Standard crouching stance for low-profile movement.

- [ ] **Animation Clip:** `anim_crouch_idle` (looping)
- [ ] **Animation Clip:** `anim_crouch_walk` (looping, 4-directional blend)
- [ ] **Transition In:** From idle/walk on crouch input
- [ ] **Transition Out:** To stand, sprint, or jump
- [ ] **Input Mapping:** `C` key or crouch button
- [ ] **Collision:** Smaller capsule when crouched

### 3. 🧗 Mount Wall
**Description:** Character pulls themselves up onto a wall ledge from below.

- [ ] **Animation Clip:** `anim_mount_wall_start` (one-shot)
- [ ] **Animation Clip:** `anim_mount_wall_loop` (looping while climbing)
- [ ] **Animation Clip:** `anim_mount_wall_top` (one-shot, reaching top)
- [ ] **Blend Space:** Wall angle variation
- [ ] **Transition In:** From ground/air when wall grab detected
- [ ] **Transition Out:** To ledge stand, drop down, or continue climbing
- [ ] **Input Mapping:** Jump button near climbable surface
- [ ] **Physics:** Custom movement mode during mount

### 4. 🏃 Mid-Crouch Wall Run
**Description:** Running along a wall in a crouched position for low-profile traversal.

- [ ] **Animation Clip:** `anim_wallrun_crouch` (looping)
  - Body horizontal, close to wall
  - Crouched leg position
  - Arms extended for balance
- [ ] **Blend Space:** Speed variation + direction (left/right wall)
- [ ] **Transition In:** From crouch state when hitting wall at speed
- [ ] **Transition Out:** To jump, drop, or regular wall-run
- [ ] **Input Mapping:** Automatic from crouch + wall contact
- [ ] **Physics:** Wall friction + forward momentum

### 5. ⚡ Shift-Jump Ready (Wall Mount Direction)
**Description:** Holding shift while wall-mounted prepares a directional jump off the wall.

- [ ] **Animation Clip:** `anim_wall_jump_ready` (looping while holding shift)
  - Character coiled/tensed on wall
  - Looking in intended jump direction
  - Slight bounce/anticipation motion
- [ ] **Transition In:** From wall-mount state when shift held
- [ ] **Transition Out:** To `anim_wall_jump` on release, or back to mount if cancelled
- [ ] **Input Mapping:** Hold `Shift` while wall-mounted + aim direction
- [ ] **Jump Execution:** On shift release → launch in aimed direction
- [ ] **Physics:** Store jump vector during ready state, apply impulse on release

---

## Animation Clips Required (Total: ~15)

| Clip Name | Type | Duration | Notes |
|-----------|------|----------|-------|
| `anim_hug_corner` | Loop | 0.8s | Corner slide |
| `anim_crouch_idle` | Loop | 1.0s | Standing crouch |
| `anim_crouch_walk_fwd` | Loop | 0.6s | Crouch walk forward |
| `anim_crouch_walk_back` | Loop | 0.7s | Crouch walk backward |
| `anim_crouch_turn_l` | One-shot | 0.3s | Quick left turn |
| `anim_crouch_turn_r` | One-shot | 0.3s | Quick right turn |
| `anim_mount_wall_start` | One-shot | 0.4s | Initial grab |
| `anim_mount_wall_loop` | Loop | 0.5s | Climbing motion |
| `anim_mount_wall_top` | One-shot | 0.6s | Pull up to ledge |
| `anim_wallrun_crouch_l` | Loop | 0.4s | Left wall crouch run |
| `anim_wallrun_crouch_r` | Loop | 0.4s | Right wall crouch run |
| `anim_wall_jump_ready` | Loop | 0.3s | Coiled anticipation |
| `anim_wall_jump_launch` | One-shot | 0.5s | Jump off wall |
| `anim_land_crouch` | One-shot | 0.4s | Land in crouch |
| `anim_stand_from_crouch` | One-shot | 0.3s | Rise to standing |

---

## Implementation Tasks

### Phase 1: Animation Assets (Week 1-2)
- [ ] Source or create all animation clips
- [ ] Import animations into Unreal Engine
- [ ] Set up retargeting if using third-party assets
- [ ] Create animation notify tracks for footstep sounds, camera shakes

### Phase 2: State Machine Setup (Week 2-3)
- [ ] Create Animation Blueprint (`ABP_ParkourCharacter`)
- [ ] Build state machine with all states and transitions
- [ ] Configure blend spaces for directional movement
- [ ] Set up transition conditions and durations

### Phase 3: Movement Integration (Week 3-4)
- [ ] Implement custom CharacterMovementComponent
- [ ] Add wall detection and collision logic
- [ ] Integrate animation states with movement modes
- [ ] Tune physics parameters (gravity, friction, jump force)

### Phase 4: Input & Polish (Week 4-5)
- [ ] Map all inputs (crouch, shift-jump ready, etc.)
- [ ] Add coyote time and jump buffering
- [ ] Implement camera transitions for wall states
- [ ] Add sound effects and particle feedback

### Phase 5: Testing & Optimization (Week 5-6)
- [ ] Playtest all animation transitions
- [ ] Profile animation performance
- [ ] Fix clipping and collision issues
- [ ] Optimize blend space calculations

---

## Technical Notes

### Animation Blueprint Variables
```cpp
// Movement state
enum class EParkourState : uint8 {
    Ground,
    Air,
    WallRun,
    WallClimb,
    CornerHug,
    Crouch,
    WallJumpReady
};

UPROPERTY(EditAnywhere)
EParkourState CurrentParkourState;

// Input flags
UPROPERTY()
bool bIsCrouching;

UPROPERTY()
bool bIsShiftHeld;

UPROPERTY()
bool bOnWall;

UPROPERTY()
FVector WallNormal;

// Blend parameters
UPROPERTY(EditAnywhere)
float MoveSpeed;

UPROPERTY(EditAnywhere)
float DirectionBlend;
```

### Transition Conditions
- **Ground → Crouch:** `bIsCrouching == true`
- **Crouch → WallRun_Crouch:** `bOnWall == true && MoveSpeed > 100`
- **WallClimb → WallJumpReady:** `bIsShiftHeld == true`
- **WallJumpReady → Air:** `bIsShiftHeld == false` (release triggers jump)

---

## Dependencies

- [ ] Character mesh with proper rigging
- [ ] Collision capsule setup for wall detection
- [ ] Enhanced Input plugin configured
- [ ] Camera system for dynamic angles during parkour

---

## References & Inspiration

- *Mirror's Edge* — Parkour movement and wall-running
- *Dishonored* — Crouch mechanics and low-profile movement
- *Titanfall 2* — Wall-jumping and corner-hugging
- *Sackboy: A Big Adventure* — Fluid animation transitions

---

## Status Legend

- `[ ]` Not started
- `[~]` In progress
- `[x]` Complete