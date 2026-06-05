# 02 — Dual-Arm Reach Task

This document describes the second milestone: a clean, IL-focused dual-arm reach environment in an empty scene with switchable keyboard/gamepad teleoperation. It builds directly on [01_lift_task.md](01_lift_task.md).

---

## 1. Overview

This document describes the implementation of a custom Isaac Lab task environment for the Trossen Mobile AI dual-arm mobile manipulator, enabling switchable keyboard/gamepad teleoperation of both arms independently. This work builds on the previously established Lift task baseline (documented in [01_lift_task.md](01_lift_task.md)) and extends it to a clean, IL-focused dual-arm reach environment in an empty scene.

The final result is a working simulation where:
- The Mobile AI robot spawns in an empty scene with no table or objects
- The robot base is anchored to the world (no physics drift)
- The left and right arms can each be controlled via IK teleoperation
- A single keyboard/gamepad toggles control between the two arms
- The environment is cleaned up for Imitation Learning data collection

---

## 2. Context — What Already Existed

The previous implementation milestone (see [01_lift_task.md](01_lift_task.md)) had established:

- `assets/mobile_ai.py` — `ArticulationCfg` for the Mobile AI robot
- `mobile_ai/lift/` — A working Lift task for the left arm only
- Task IDs registered: `Isaac-Lift-Cube-MobileAI-v0`, `Isaac-Lift-Cube-MobileAI-IK-Rel-v0`
- Validated with `teleop_se3_agent.py` — left arm lift with keyboard

This session created a new **Reach task** in an empty scene as a cleaner foundation for dual-arm IL data collection.

---

## 3. Relevant File Structure

Only relevant source files are listed below. `__pycache__` directories and `.pyc` files are omitted.

```
trossen_ai_isaac/
├── assets/robots/
│   └── mobile_ai/
│       └── mobile_ai.usd                          # Robot USD asset (unchanged)
│
├── scripts/
│   └── teleoperation/
│       ├── teleop_se3_agent.py                    # Original teleop script (unchanged)
│       └── teleop_dual_arm_switch.py              # NEW — switchable dual-arm teleop
│
└── source/trossen_ai_isaac/trossen_ai_isaac/
    └── tasks/manager_based/manipulation/
        ├── assets/
        │   ├── __init__.py                        # MODIFIED — added mobile_ai import
        │   └── mobile_ai.py                       # Robot ArticulationCfg (unchanged)
        │
        └── mobile_ai/
            ├── __init__.py                        # MODIFIED — imports lift + reach
            ├── lift/                              # Pre-existing (unchanged)
            │   ├── __init__.py
            │   └── config/
            │       ├── __init__.py                # Registers Lift task IDs
            │       ├── joint_pos_env_cfg.py
            │       └── ik_rel_env_cfg.py
            └── reach/                             # NEW
                ├── __init__.py                    # NEW — registers Reach task IDs
                ├── config/
                │   └── __init__.py                # NEW — package marker
                └── reach_env_cfg.py               # NEW — full IL environment config
```

---

## 4. Registered Task IDs

| Task ID | Config Class | Use Case |
|---|---|---|
| `Isaac-Reach-MobileAI-IK-Rel-v0` | `MobileAIReachEnvCfg` | IL training (multi-env) |
| `Isaac-Reach-MobileAI-IK-Rel-Play-v0` | `MobileAIReachEnvCfg_PLAY` | Teleoperation (1 env) |

**Launch command:**
```bash
cd ~/trossen_ai_isaac && ~/IsaacLab/isaaclab.sh -p scripts/teleoperation/teleop_dual_arm_switch.py \
    --task Isaac-Reach-MobileAI-IK-Rel-Play-v0 \
    --teleop_device keyboard
```

---

## 5. Environment Design — `reach_env_cfg.py`

Full file located at:
`source/trossen_ai_isaac/trossen_ai_isaac/tasks/manager_based/manipulation/mobile_ai/reach/reach_env_cfg.py`

### 5.1 Scene — `MobileAIReachSceneCfg`

Minimal empty scene: robot + ground plane + dome light. No table, no cube, no distractors. The robot uses `MOBILE_AI_HIGH_PD_CFG` (gravity disabled on all links — standard for IK control) with `fix_root_link=True` added to anchor the base to the world frame.

### 5.2 Actions — `ActionsCfg`

Both arms use `DifferentialInverseKinematicsActionCfg` in relative pose mode (`use_relative_mode=True`, `ik_method="dls"`).

| Parameter | Left Arm | Right Arm |
|---|---|---|
| `joint_names` | `follower_left_joint_[0-5]` | `follower_right_joint_[0-5]` |
| `body_name` (EE) | `follower_left_link_6` | `follower_right_link_6` |
| `scale` | `0.05` | `0.05` |

Total action space: **12D** — indices `[0:6]` = left arm, `[6:12]` = right arm.

**Why `ik_method="dls"`:** Damped Least Squares avoids singularity blow-up near joint limits.  
**Why `use_relative_mode=True`:** Each input is a delta from the current EE pose — natural for human teleoperation.

### 5.3 Commands — `CommandsCfg`

Both arms have a `UniformPoseCommandCfg` that provides the goal pose as part of the observation vector (required for IL policy input). `debug_vis=False` hides the floating target markers during teleoperation.

| Command name | Tracks |
|---|---|
| `ee_pose_left` | `follower_left_link_6` |
| `ee_pose_right` | `follower_right_link_6` |

### 5.4 Observations — `ObservationsCfg.PolicyCfg`

The full observation vector concatenated in this order:

| Term | Content | Shape |
|---|---|---|
| `joint_pos` | All joint positions (relative) | `(26,)` |
| `joint_vel` | All joint velocities (relative) | `(26,)` |
| `pose_command_left` | Left arm goal pose | `(7,)` |
| `pose_command_right` | Right arm goal pose | `(7,)` |
| `actions` | Last sent action | `(12,)` |

### 5.5 Rewards & Terminations — Placeholders

`ManagerBasedRLEnvCfg` requires `rewards` and `terminations` fields even for IL-only environments. Both are empty `@configclass` placeholders. **Rewards are not used in Imitation Learning.**

### 5.6 Teleop Device Configuration

| Device | `pos_sensitivity` | `rot_sensitivity` | Notes |
|---|---|---|---|
| Keyboard | `0.4` | `0.8` | Default — tuned from initial `0.05` |
| Gamepad | `1.0` | `1.6` | Default |
| SpaceMouse | `0.4` | `0.8` | Default |

Effective movement per step: `scale × pos_sensitivity = 0.05 × 0.4 = 0.02 m/step`

---

## 6. Dual-Arm Switch Teleop Script — `teleop_dual_arm_switch.py`

Full file located at: `scripts/teleoperation/teleop_dual_arm_switch.py`

### Architecture

The script implements a device-agnostic action router. The environment always receives a 12D action vector. The active arm slot receives the 6D delta; the inactive arm slot is zeroed.

```
Input device (6D delta) → router → [left_6D | right_6D] 12D → env.step()
```

This design is future-proof: VR dual controllers can fill both slots simultaneously without changing the environment config.

### Arm Toggle Keys

| Device | Key | Action |
|---|---|---|
| Keyboard | `TAB` | Toggle active arm LEFT ↔ RIGHT |
| Gamepad | `Y` button | Toggle active arm LEFT ↔ RIGHT |
| Both | `R` | Reset environment |

### All Keyboard Controls

| Key | Action |
|---|---|
| `W / S` | EE forward / back |
| `A / D` | EE left / right |
| `Q / E` | EE up / down |
| `Z / X` | Rotate EE |
| `T / G` | Rotate EE |
| `C / V` | Rotate EE |
| `TAB` | Toggle active arm |
| `R` | Reset environment |

---

## 7. Bugs Fixed & Issues Resolved

### Bug 1 — Robot Base Flying Around

**Symptom:** Arm movement caused the robot base to fly off into the air.  
**Root cause:** `MOBILE_AI_HIGH_PD_CFG` disables gravity on the entire articulation including the base. Without an anchor, arm reaction forces pushed the base away.  
**Fix:** `fix_root_link=True` in `ArticulationRootPropertiesCfg` at spawn time.  
**Failed attempt:** Setting `fix_base_link` in `__post_init__` caused error: `Attribute 'FixBaseLink' does not exist on prim '/World/envs/env_0/Robot/base_footprint'` — the correct field is `fix_root_link` and it must be set via `ArticulationRootPropertiesCfg`, not post-init.

---

### Bug 2 — Arm Moving Too Fast

**Symptom:** Arm overshot and jumped erratically with keyboard input.  
**Root cause:** Default `scale=0.5` — 50× larger than WXAI reference value of `0.01`.  
**Fix:** Reduce `scale` to `0.05`. Sensitivities kept at default values.

---

### Bug 3 — `rewards` / `terminations` MISSING Error

**Symptom:** `Failed to create environment: Missing values detected for fields: rewards, terminations`  
**Root cause:** `ManagerBasedRLEnvCfg` declares these as required even if unused.  
**Fix:** Add empty `@configclass` placeholders for both and assign them in `MobileAIReachEnvCfg`.

---

### Issue 4 — Floating Coordinate Markers on Reset

**Symptom:** Two RGB axis gizmos appeared at random positions on every environment reset.  
**Root cause:** `debug_vis=True` on `UniformPoseCommandCfg` draws a randomly sampled goal pose marker.  
**Fix:** `debug_vis=False` on both `ee_pose_left` and `ee_pose_right`.

---

### Issue 5 — EE Frame Gizmos Not Showing

**Symptom:** After disabling command visuals, the arm-tip coordinate frames disappeared.  
**Root cause:** EE frames come from the IK action visualizer, not the command. It only renders while simulation is stepping AND the Actions checkbox is enabled in the UI.  
**Fix:** Set `debug_vis=True` on both arm actions + enable **Actions** checkbox in Isaac Sim → IsaacLab tab → Scene Debug Visualization.

---

## 8. Key Design Decisions

| Decision | Rationale |
|---|---|
| Use `MOBILE_AI_HIGH_PD_CFG` | Gravity disabled = IK tracks position without fighting gravity. Using gravity-enabled config requires explicit compensation torques. |
| Two separate `ActionTermCfg` (not one bimanual) | `DifferentialInverseKinematicsActionCfg` operates on one kinematic chain. Separate terms allow independent IK per arm. |
| `scale` separate from `pos_sensitivity` | `scale` caps max EE delta per sim step (IK stability). `pos_sensitivity` scales device input (feel). Tuning them independently avoids IK instability. |
| Keep `CommandsCfg` for IL | The IL policy needs the goal pose in its observation at every step. Removing commands would make the policy blind to the task objective. |
| Empty `RewardsCfg` / `TerminationsCfg` | No `ManagerBasedILEnvCfg` exists in Isaac Lab. The RL base class is used for all manager-based envs — placeholders satisfy the interface without computing anything. |

---

## 9. Current Status

| Feature | Status |
|---|---|
| Mobile AI robot in empty scene | ✅ |
| Robot base fixed to world | ✅ |
| Left arm IK teleoperation | ✅ |
| Right arm IK teleoperation | ✅ |
| TAB / Y button arm toggle | ✅ |
| Clean IL-focused config | ✅ |
| Registered Gymnasium task IDs | ✅ |
| **Arm drift (both arms drift slowly)** | ⚠️ See [03_arm_drift.md](03_arm_drift.md) |

---

## 10. Next Steps

| Step | Priority | Description |
|---|---|---|
| Fix arm drift | High | Both arms drift slowly without input — IK integrator issue (see [03_arm_drift.md](03_arm_drift.md)) |
| Define IL task | High | Pick-place or bimanual handover — determines object to add |
| Add object to scene | High | `RigidObjectCfg` in `MobileAIReachSceneCfg` |
| Data collection setup | High | `RecorderManager` HDF5 or LeRobot integration |
| Gripper control | Medium | Dedicated key callback per arm in teleop script |
| VR dual-arm teleop | Later | `teleop_vr_dual_arm.py` — same env config, both arms simultaneously |
| Sim-to-Real | Later | Domain randomization on object poses, lighting, joint friction |
