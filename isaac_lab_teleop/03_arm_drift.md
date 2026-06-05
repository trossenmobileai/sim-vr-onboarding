# 03 — Arm Drift Investigation

Both arms in the dual-arm reach environment (see [02_dual_arm_reach.md](02_dual_arm_reach.md)) drift slowly even when no input is given. This document explains the investigation, root cause, and conclusion.

---

## Summary

In our Mobile AI dual-arm teleoperation setup, the robot arms **slowly drift** in Isaac Lab even when we send perfectly zero actions every frame.

We verified that this drift:
- Happens **without any teleop device activity**.
- Appears even in a **headless debug script** that sends all-zero actions.
- Persists with gravity disabled and high PD gains on the arms.

Therefore, the drift is not a bug in our teleop script. It comes from how Isaac Sim / PhysX and the IK controller behave for this robot model.

---

## 1. What we measured

We created a small debug script that:

- Loads the task `Isaac-Reach-MobileAI-IK-Rel-Play-v0`.
- Resets the environment once.
- For 30 simulation steps, sends an action of all zeros with shape `(1, 12)` (6 DoF per arm).
- Prints the left end-effector (EE) world position every 5 steps.

Example output:

```text
Step 00: [0.4269017, 0.2999096, 1.2160472]
Step 05: [0.4268934, 0.2999928, 1.2160871]
Step 10: [0.4268849, 0.3000801, 1.2161233]
Step 15: [0.4268762, 0.3001674, 1.2161595]
Step 20: [0.4268673, 0.3002547, 1.2161951]
Step 25: [0.4268582, 0.3003421, 1.2162306]
```

Even though every action is exactly zero, the EE position slowly changes over time:

- Y increases from `0.2999` to `0.3003` (about 0.00043 over 25 steps).
- Z increases from `1.2160` to `1.2162` (about 0.00018 over 25 steps).

This proves that the robot drifts **even when we are not sending any commands at all**.

---

## 2. How control is configured

The configurations below are set up in `02_dual_arm_reach.md` §5.

### Robot asset configuration (`mobile_ai.py`)

We use an Isaac Lab `ArticulationCfg` called `MOBILE_AI_HIGH_PD_CFG`:

- It is copied from a base config `MOBILE_AI_CFG`.
- It explicitly disables gravity for the robot:
  ```python
  MOBILE_AI_HIGH_PD_CFG.spawn.rigid_props.disable_gravity = True
  ```
- It sets high PD gains for both arms:
  ```python
  MOBILE_AI_HIGH_PD_CFG.actuators["left_arm"].stiffness  = 400.0
  MOBILE_AI_HIGH_PD_CFG.actuators["left_arm"].damping    = 80.0
  MOBILE_AI_HIGH_PD_CFG.actuators["right_arm"].stiffness = 400.0
  MOBILE_AI_HIGH_PD_CFG.actuators["right_arm"].damping   = 80.0
  ```

So the arms should be held relatively stiff, and gravity should not pull them down.

### Environment configuration (`reach_env_cfg.py`)

In the `MobileAIReachSceneCfg` we use that robot configuration:

```python
robot: ArticulationCfg = MOBILE_AI_HIGH_PD_CFG.replace(
    prim_path="{ENV_REGEX_NS}/Robot",
    spawn=MOBILE_AI_HIGH_PD_CFG.spawn.replace(
        rigid_props=sim_utils.RigidBodyPropertiesCfg(
            disable_gravity=True,
            max_depenetration_velocity=5.0,
        ),
        articulation_props=sim_utils.ArticulationRootPropertiesCfg(
            enabled_self_collisions=True,
            solver_position_iteration_count=16,
            solver_velocity_iteration_count=0,
            fix_root_link=True,
        )
    ),
)
```

We also configure the actions to use **relative IK in SE(3)**, 6D per arm:

```python
self.actions.left_arm_action = DifferentialInverseKinematicsActionCfg(
    asset_name="robot",
    joint_names=["follower_left_joint_[0-5]"],
    body_name="follower_left_link_6",
    controller=DifferentialIKControllerCfg(
        command_type="pose",
        use_relative_mode=True,
        ik_method="pinv",
    ),
    scale=0.05,
)
# same for right_arm_action
```

Key points:

- `command_type="pose"` and `use_relative_mode=True` mean each 6D action is a **pose delta** (linear + angular twist) in end-effector space.
- A 6D vector of all zeros means: *"do not move the end-effector this step"*.

---

## 3. Teleop script behavior

In `teleop_dual_arm_switch.py` we:

- Use a 12D action: `[left_6D, right_6D]`.
- Define slices:
  ```python
  ARM_SLICE = {
      LEFT_ARM:  slice(0, 6),
      RIGHT_ARM: slice(6, 12),
  }
  ```
- On every frame, we build a 12D action and call `env.step(actions)`.
- When no keys are pressed, we explicitly set both arms' actions to zero.

We added debug prints of `full_action` every 60 frames. While not touching any input, we saw:

```text
[DEBUG] step 0,   full_action = [0.0, ..., 0.0]
[DEBUG] step 60,  full_action = [0.0, ..., 0.0]
[DEBUG] step 120, full_action = [0.0, ..., 0.0]
...
```

So the teleop script is **correctly sending all zeros** when we are idle, and it always calls `env.step(actions)` every frame.

Despite this, the arms still drift in the GUI over time, matching the drift observed in the headless debug script.

---

## 4. Why the arms still drift

Because we have:

- Gravity disabled for the robot.
- High PD gains on the arm actuators.
- Relative IK with zero pose deltas on idle.
- All-zero actions confirmed in both teleop and debug scripts.

The remaining drift must come from the **physics solver and the robot model**, not from our control logic.

Likely contributors:

1. **Physics integration and constraints**  
   - PhysX integrates joint states and resolves constraints at each time step.  
   - With a complex articulated system (mobile base + 2 arms + grippers), small numerical errors can accumulate.  
   - Even with gravity off, joints, limits, and contact constraints can cause small residual motions.

2. **Differential IK and PD interaction**  
   - The differential IK controller computes joint velocities that try to keep the EE pose constant when deltas are zero, but it's still an iterative numerical method.  
   - PD actuators try to hold joint positions, but do not completely eliminate tiny velocities or inconsistencies in the solution.

3. **Initial robot pose and joint limits**  
   - The initial joint configuration may not be a perfectly "balanced" configuration for this robot.  
   - The solver may slowly adjust the configuration toward a nearby state that better satisfies all constraints and internal preferences.

We confirmed that this drift exists **even in a minimal script with zero actions and no teleoperation**, which means:

> The drift is a simulator-level artifact for this robot and controller combination, not a bug in our teleop script.

---

## 5. Why we cannot fully eliminate the drift (in this project)

In principle, one could try to eliminate drift by:

- Re-tuning the robot USD/URDF model and all its constraints.
- Modifying the internal behavior of the IK controller or PhysX solver.
- Implementing a custom stabilizer that continually corrects tiny pose errors.

However, these are **out of scope** for our project because they require deep changes to Isaac Sim internals and the robot asset, not just our task or teleop code.

Within our control, we can only:

- Increase PD gains further (e.g. stiffness 800, damping 160) to make the arm stiffer, which may reduce drift but can make motion more "snappy".
- Slightly adjust initial joint positions to find a configuration that drifts less.
- Accept the small drift as a characteristic of the simulator and design around it (e.g. by focusing on short teleop episodes).

Given the scope of our project (VR teleoperation and imitation learning with the Mobile AI platform), the most practical decision is to:

- Keep the current IK-Rel setup and teleop script (which are logically correct).
- Accept the small simulation drift as an inherent numerical effect in Isaac Lab for this robot.

---

## 6. How to explain this to future team members

When someone new notices the arms drifting in sim, we can explain:

> We already investigated this thoroughly. The arms drift even when we send exact zero actions in a headless debug script, with gravity disabled and high PD gains. The drift comes from the underlying physics and IK solver for this robot in Isaac Lab, not from our teleop script. We use relative IK with zero deltas on idle and always step the environment every frame. Fixing the drift completely would require changing Isaac Sim internals or the robot asset, which is out of scope for our project. For our purposes (VR teleoperation and data collection), we accept this small drift.

This makes clear that:

- The behavior is understood and documented.
- It is not caused by an error in our code.
- A "perfect" fix is beyond the project's control.
