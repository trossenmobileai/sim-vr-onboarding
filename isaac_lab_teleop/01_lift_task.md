# 01 — Lift Task: Isaac Lab Integration

This is the foundational milestone for teleoperating the Mobile AI robot in Isaac Lab. It establishes the Gymnasium task registration chain, the robot asset config, and a working single-arm lift environment. The next milestone builds directly on this — see [02_dual_arm_reach.md](02_dual_arm_reach.md).

---

## 1. Overview

This document describes the implementation of Isaac Lab task configuration files required to enable gamepad/VR teleoperation of the Trossen Mobile AI dual-arm mobile manipulator inside NVIDIA Isaac Sim.

The goal was to register the Mobile AI robot as a valid Isaac Lab Gym environment, so that the existing `teleop_se3_agent.py` script can be launched with:

```bash
python teleop_se3_agent.py \
    --task Isaac-Lift-Cube-MobileAI-IK-Rel-v0 \
    --teleop_device gamepad
```

This is the first step toward collecting tele-operated demonstrations for Imitation Learning (Diffusion Policy / ACT).

---

## 2. Background — How Isaac Lab Task Registration Works

Isaac Lab uses the [Gymnasium](https://gymnasium.farama.org/) registry system to manage simulation environments. Every robot task must be explicitly registered with a unique string ID (e.g. `Isaac-Lift-Cube-WXAI-v0`) before it can be loaded by any training or teleoperation script.

The registration chain works as follows:

```
tasks/__init__.py
  └── import_packages()          # auto-discovers all sub-packages
        └── mobile_ai/__init__.py
              └── lift/__init__.py
                    └── config/__init__.py
                          └── gym.register("Isaac-Lift-Cube-MobileAI-IK-Rel-v0")
                                └── points to → ik_rel_env_cfg.py
                                                  └── inherits from joint_pos_env_cfg.py
                                                        └── uses → MOBILE_AI_CFG (robot asset)
                                                                      └── loads → mobile_ai.usd
```

Every layer in this chain must exist as a Python package (`__init__.py`) for the auto-discovery to work. A missing file at any level silently breaks the chain and the task ID never gets registered.

---

## 3. Baseline — WXAI Integration

Trossen Robotics provided a fully integrated Isaac Lab task for the **WXAI single-arm robot**. This served as the direct template for the Mobile AI integration.

The WXAI file structure is:

```
tasks/manager_based/manipulation/
├── assets/
│   ├── __init__.py              # exports WXAI_BASE_CFG
│   └── wxai.py                  # ArticulationCfg for WXAI
└── wxai/
    ├── __init__.py
    └── lift/
        ├── __init__.py
        └── config/
            ├── __init__.py              # registers Isaac-Lift-Cube-WXAI-* task IDs
            ├── joint_pos_env_cfg.py     # joint position control environment
            ├── ik_rel_env_cfg.py        # relative IK control (used for teleop)
            ├── ik_abs_env_cfg.py        # absolute IK control
            └── agents/
                ├── __init__.py
                └── rsl_rl_ppo_cfg.py    # PPO hyperparameters
```

The WXAI robot is a **single 6-DOF arm** with one gripper. The Mobile AI platform differs in that it has:
- Two 6-DOF arms (left + right follower arms)
- Two grippers
- A differential drive mobile base (left/right wheels)
- A different end-effector link naming convention

Because of these differences, the WXAI files cannot be reused directly — a parallel Mobile AI task structure had to be created.

---

## 4. Why Each File Is Needed

### 4.1 `assets/mobile_ai.py` — Robot Asset Configuration

**What it is:** An `ArticulationCfg` object that tells Isaac Lab how to spawn the Mobile AI robot in simulation.

**Why it is needed:** Isaac Lab does not load USD files directly. It requires an `ArticulationCfg` that specifies:
- The path to the USD file (`mobile_ai.usd`)
- Physics properties (gravity, solver iterations, self-collision)
- The initial joint positions (home pose)
- The actuator groups — which joints are controlled and how

Without this file, Isaac Lab has no knowledge of the Mobile AI robot and cannot spawn it in any environment.

**Why it cannot be shared with WXAI:** The WXAI config (`wxai.py`) references `joint_[0-5]` and `left_carriage_joint` — joint names specific to the WXAI single arm. The Mobile AI joints are named `follower_left_joint_[0-5]`, `follower_right_joint_[0-5]`, `follower_left_left_carriage_joint`, etc. Using the wrong joint names causes Isaac Lab to fail silently or raise a runtime error when it tries to drive joints that do not exist in the USD.

**Key design decision:** Only `follower_left_left_carriage_joint` (and right equivalent) is listed as an actuator — not `follower_left_right_carriage_joint`. This is because in the USD file, the right carriage joint is defined as a **mimic joint**, meaning it automatically mirrors the left carriage. Commanding both would cause a conflict.

---

### 4.2 `assets/__init__.py` — Asset Package Export

**What it is:** The package init file for the `assets/` folder.

**Why it is needed:** Python requires an `__init__.py` to treat a directory as a package. The existing file already exported `WXAI_BASE_CFG`. Adding `from .mobile_ai import *` makes `MOBILE_AI_CFG` importable by any environment config file using:

```python
from trossen_ai_isaac.tasks.manager_based.manipulation.assets import MOBILE_AI_CFG
```

---

### 4.3 `mobile_ai/lift/config/joint_pos_env_cfg.py` — Joint Position Environment

**What it is:** A `@configclass` that defines the complete lift task environment for Mobile AI under joint position control.

**Why it is needed:** This file is the heart of the task definition. It tells Isaac Lab:
- Which robot to use (`MOBILE_AI_CFG`)
- What the action space is (joint position commands to left arm + gripper)
- What the observation target is (end-effector pose of `follower_left_ee_gripper_link`)
- Where the object (cube) spawns and what it looks like
- What workspace region the robot should reach into
- How to track the end-effector frame for reward computation

**Why the left arm was chosen as primary:** The standard Isaac Lab `LiftEnvCfg` is designed around a single arm with one `arm_action` and one `gripper_action`. Extending to true bimanual control requires a custom reward structure and dual action spaces, which is a later milestone. Starting with the left arm as the primary teleoperated arm gives a working baseline that is directly comparable to WXAI.

**Key values sourced from USD inspection:**

| Parameter | Value | Source |
|-----------|-------|--------|
| `joint_names` (arm) | `follower_left_joint_[0-5]` | USD joint dump |
| `joint_names` (gripper) | `follower_left_left_carriage_joint` | USD joint dump |
| `body_name` (EE) | `follower_left_ee_gripper_link` | USD stage tree |
| `prim_path` (EE frame) | `{ENV_REGEX_NS}/Robot/follower_left_ee_gripper_link` | USD stage tree |
| Gripper open value | `0.044` m | To be verified against URDF joint limit |

---

### 4.4 `mobile_ai/lift/config/ik_rel_env_cfg.py` — Relative IK Environment (Teleop)

**What it is:** A child class of `MobileAICubeLiftEnvCfg` that replaces the joint position action with a **Differential Inverse Kinematics (IK)** action in relative pose mode.

**Why it is needed:** The `teleop_se3_agent.py` script reads 6D delta commands from the gamepad (3D position delta + 3D rotation delta) and passes them directly as the environment action. This only works when the environment action space is a **6D Cartesian delta** — not joint positions.

The joint position environment (`joint_pos_env_cfg.py`) is designed for RL training where the policy outputs joint angles. The IK relative environment is designed for teleoperation where a human outputs Cartesian deltas via a gamepad or SpaceMouse.

**Why `use_relative_mode=True`:** Relative mode means each gamepad input is interpreted as a *delta* from the current EE pose. Absolute mode would require the user to command exact world-frame poses, which is impractical for a human operator.

**Why `ik_method="dls"`:** Damped Least Squares (DLS) is the standard numerically stable IK solver for redundant manipulators. It avoids singularity blow-up near joint limits.

**Why `gripper_action` is removed:** In relative IK mode, the action vector is exactly 6D (position + rotation delta). Adding a 7th gripper dimension would break the `teleop_se3_agent.py` action pipeline, which expects a fixed-size 6D vector from the gamepad. The gripper is instead controlled by a dedicated button callback registered separately in the teleop script.

---

### 4.5 `mobile_ai/lift/config/__init__.py` — Gym Task Registration

**What it is:** The file that registers the Mobile AI task IDs in the Gymnasium registry.

**Why it is needed:** Without `gym.register()` calls, the task string `Isaac-Lift-Cube-MobileAI-IK-Rel-v0` does not exist. Any call to `gym.make("Isaac-Lift-Cube-MobileAI-IK-Rel-v0")` would raise a `gymnasium.error.UnregisteredEnv` exception. The task ID is the public interface — it is the only thing `teleop_se3_agent.py` needs to know to load the entire environment configuration chain.

**Registered task IDs:**

| Task ID | Config class | Use case |
|---------|-------------|----------|
| `Isaac-Lift-Cube-MobileAI-v0` | `MobileAICubeLiftEnvCfg` | PPO training |
| `Isaac-Lift-Cube-MobileAI-Play-v0` | `MobileAICubeLiftEnvCfg_PLAY` | Policy evaluation |
| `Isaac-Lift-Cube-MobileAI-IK-Rel-v0` | `MobileAICubeLiftEnvCfg` (IK) | Gamepad/VR teleop |

Note: `IK-Abs` was not registered because `ik_abs_env_cfg.py` has not been created yet. It can be added as a future task.

---

### 4.6 `agents/rsl_rl_ppo_cfg.py` — PPO Hyperparameters

**What it is:** RSL-RL runner configuration for PPO training.

**Why it is needed:** The `gym.register()` call requires both an environment config entry point and an agent config entry point. Even for teleoperation (which does not use PPO), the registry requires this field to be valid. The file was copied directly from WXAI because the PPO hyperparameters are task-agnostic for a standard lift task.

---

### 4.7 `mobile_ai/__init__.py` and `mobile_ai/lift/__init__.py` — Package Init Files

**What they are:** Empty-ish Python package markers.

**Why they are needed:** Python's `import_packages()` utility used in `tasks/__init__.py` traverses the directory tree looking for sub-packages. A directory without an `__init__.py` is not a Python package and is invisible to the import system. Without these two files, the entire `mobile_ai/` tree would be ignored and no task IDs would be registered.

The `mobile_ai/__init__.py` was carefully edited to only import `from .lift import *` — removing the WXAI references to `cabinet` and `reach` which do not exist for Mobile AI yet. Keeping those imports would cause an `ImportError` at startup.

---

## 5. File Structure Created

```
tasks/manager_based/manipulation/
├── assets/
│   ├── __init__.py              # MODIFIED: added "from .mobile_ai import *"
│   └── mobile_ai.py             # NEW: MOBILE_AI_CFG, MOBILE_AI_HIGH_PD_CFG
└── mobile_ai/
    ├── __init__.py              # NEW: package marker, imports lift
    └── lift/
        ├── __init__.py          # NEW: imports config
        └── config/
            ├── __init__.py              # NEW: registers 3 Gym task IDs
            ├── joint_pos_env_cfg.py     # NEW: joint position lift environment
            ├── ik_rel_env_cfg.py        # NEW: relative IK lift environment (teleop)
            └── agents/
                ├── __init__.py          # NEW: copied from WXAI
                └── rsl_rl_ppo_cfg.py    # NEW: copied from WXAI
```

**Total:** 8 new files, 1 modified file.

---

## 6. What Was Not Changed

- `teleop_se3_agent.py` — no modifications needed; it loads any registered task by name
- `tasks/__init__.py` — no modifications needed; `import_packages()` auto-discovers Mobile AI
- The Mobile AI USD file — no modifications; the USD was already exported from Isaac Sim
- WXAI files — untouched; Mobile AI runs in parallel, not as a replacement

---

## 7. Pending Verification

The following items need to be verified when GPU resources are available:

1. **Gripper open value** (`0.044` m): Verify against the URDF joint limit for `follower_left_left_carriage_joint`:

   ```bash
   grep -A5 "left_carriage_joint" <path_to_mobile_ai.urdf>
   ```

2. **End-effector link name**: Confirm `follower_left_ee_gripper_link` exists in the USD stage at runtime. If it does not, check the exact link name in Isaac Sim Stage panel.

3. **Full launch test**:

   ```bash
   cd /home/trossen-admin/trossen_ai_isaac
   /home/trossen-admin/isaacsim/python.sh \
       source/trossen_ai_isaac/scripts/teleop_se3_agent.py \
       --task Isaac-Lift-Cube-MobileAI-IK-Rel-v0 \
       --teleop_device gamepad \
       --num_envs 1
   ```

---

## 8. Next Steps

Once teleoperation is verified:

1. **Collect demonstrations** using `teleop_se3_agent.py` and record to HDF5 using LeRobot or Isaac Lab's data collection utilities
2. **Train Diffusion Policy** on the collected demonstrations
3. **Extend to dual-arm** by adding a second `arm_action` for the right arm and customizing the reward function — see [02_dual_arm_reach.md](02_dual_arm_reach.md)
4. **Add base mobility** by mapping gamepad joystick to `left_wheel` / `right_wheel` velocity commands via a custom action term
5. **Sim-to-Real transfer** using domain randomization on object poses, lighting, and joint friction
