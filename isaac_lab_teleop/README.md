# Isaac Lab Teleop — Implementation Notes

This folder contains deep-dive implementation documentation for the Isaac Lab task environments and teleoperation scripts developed for the Trossen Mobile AI platform. These docs record the *how* and *why* of each milestone in detail.

They complement the onboarding flow in the [parent repository](../README.md) — read the main 01–07 guides first, then come here for implementation depth.

---

## Contents

| Doc | What It Covers |
|-----|----------------|
| [`01_lift_task.md`](01_lift_task.md) | Foundational milestone: Gymnasium task registration chain, robot asset config (`ArticulationCfg`), and working single-arm Lift environment for keyboard/gamepad teleop |
| [`02_dual_arm_reach.md`](02_dual_arm_reach.md) | Second milestone: dual-arm Reach environment in an empty scene, switchable arm teleop script (`teleop_dual_arm_switch.py`), and all bugs fixed along the way |
| [`03_arm_drift.md`](03_arm_drift.md) | Investigation of slow arm drift observed in the Reach environment — root cause analysis, measurements, and why a full fix is out of scope |

Read them in order — each doc assumes the previous one.

---

## Relationship to Main Docs

- [`../06_vr_teleoperation.md`](../06_vr_teleoperation.md) — explains how VR input plugs into the same task framework described here
- [`../07_troubleshooting.md`](../07_troubleshooting.md) — S7 entry links here for the arm drift symptom
