# 06 — VR Teleoperation

This document explains how VR hand-tracking plugs into the Isaac Lab task framework to control the robot arms, starting from a working VR session (see [`05_vr_setup.md`](05_vr_setup.md)).

---

## How VR Teleop Fits the Existing Framework

VR teleoperation reuses the **exact same Isaac Lab task environments** as keyboard and gamepad teleop — there is no separate VR-only environment. The only difference is the input device: instead of a `Se3Keyboard` or `Se3Gamepad`, the script uses an `OpenXRDevice` that reads hand-tracking data from the Quest 3 via SteamVR.

```
Meta Quest 3 hand tracking
        │
        ▼
ALVR → SteamVR → OpenXR API
        │
        ▼
Isaac Lab OpenXRDevice
        │
        ▼  6D delta per hand (position + rotation)
Action router (teleop script)
        │
        ▼  12D action [left_6D | right_6D]
Isaac Lab task environment (Reach env)
        │
        ▼
Trossen arm IK → joint commands → Isaac Sim robot
```

The 12D action vector and the `Isaac-Reach-MobileAI-IK-Rel-Play-v0` task environment are the same ones used in keyboard teleop — see [`isaac_lab_teleop/02_dual_arm_reach.md`](isaac_lab_teleop/02_dual_arm_reach.md) §4–5 for full details.

---

## Prerequisites

Before running a VR teleop session you need:

1. **A working VR session** — ALVR streaming to the Quest 3, SteamVR running, Quest 3 showing the SteamVR home. Follow the full checklist in [`05_vr_setup.md`](05_vr_setup.md).
2. **Isaac Lab task environments registered** — the `trossen_ai_isaac` extension loaded in Isaac Lab. Background in [`isaac_lab_teleop/01_lift_task.md`](isaac_lab_teleop/01_lift_task.md).
3. **Dual-arm Reach environment** — the `Isaac-Reach-MobileAI-IK-Rel-Play-v0` task ID must be registered. Covered in [`isaac_lab_teleop/02_dual_arm_reach.md`](isaac_lab_teleop/02_dual_arm_reach.md).

---

## Current State

| Capability | Status |
|---|---|
| Keyboard/gamepad dual-arm teleop | ✅ Working (`teleop_dual_arm_switch.py`) |
| VR single-arm teleop (Franka reference) | ✅ Working (Isaac Lab built-in) |
| VR dual-arm teleop for Mobile AI | 🔲 Roadmap — `teleop_vr_dual_arm.py` |

The dedicated VR dual-arm teleop script (`teleop_vr_dual_arm.py`) is the next step after keyboard teleop is fully validated. It will use the same `Isaac-Reach-MobileAI-IK-Rel-Play-v0` environment and swap the input device from `Se3Keyboard` to `OpenXRDevice`. This page will be updated when that script lands.

---

## Validating Hand Tracking (Franka Reference Task)

While the Mobile AI VR script is pending, you can validate that the full VR stack works end-to-end using Isaac Lab's built-in Franka example:

```bash
cd ~/IsaacLab
./isaaclab.sh -p source/standalone/environments/teleoperation/teleop_se3_agent.py \
    --task Isaac-Stack-Cube-Franka-IK-Rel-v0 \
    --teleop_device openxr \
    --num_envs 1
```

Expected: the Franka arm in the Isaac Sim viewport follows your right hand's pose in VR. Press **S** to start arm tracking.

---

## Known Issue — Arm Drift

Once inside any Isaac Lab teleop session (VR or keyboard), both arms will drift slowly even when no input is sent. This is a simulator-level numerical artifact — not a bug in the teleop script. It was investigated thoroughly and is documented in [`isaac_lab_teleop/03_arm_drift.md`](isaac_lab_teleop/03_arm_drift.md).

**Short version:** drift comes from PhysX constraint solving on this robot model. It is accepted as an inherent characteristic for the scope of this project.

---

## Further Reading

| Topic | Doc |
|---|---|
| VR stack setup (ALVR, SteamVR, Quest 3) | [`05_vr_setup.md`](05_vr_setup.md) |
| Task registration and Gymnasium mechanics | [`isaac_lab_teleop/01_lift_task.md`](isaac_lab_teleop/01_lift_task.md) |
| Dual-arm Reach env and teleop script | [`isaac_lab_teleop/02_dual_arm_reach.md`](isaac_lab_teleop/02_dual_arm_reach.md) |
| Arm drift root cause | [`isaac_lab_teleop/03_arm_drift.md`](isaac_lab_teleop/03_arm_drift.md) |
| All known issues | [`07_troubleshooting.md`](07_troubleshooting.md) |
