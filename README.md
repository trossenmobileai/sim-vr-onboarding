# Simulation & VR Onboarding

Welcome to the Simulation & VR sub-team. This documentation covers everything you need to understand and contribute to the Isaac Sim simulation and VR teleoperation work done so far on the Trossen Mobile AI project.

> **Prerequisites assumed:** ROS2 Humble basics (topics, nodes, rosbag). No prior Isaac Sim experience required.

---

## How to Read This

Go through the documents in order. Each builds on the previous one.

| Step | File                                               | What You'll Learn                                               |
| ---- | -------------------------------------------------- | --------------------------------------------------------------- |
| 1    | [`01_project_overview.md`](01_project_overview.md) | Where sim/VR fits in the full project pipeline                  |
| 2    | [`02_isaac_sim_basics.md`](02_isaac_sim_basics.md) | Core Isaac Sim concepts (USD, Action Graph, PhysX, ROS2 bridge) |
| 3    | [`03_simulation_setup.md`](03_simulation_setup.md) | Our custom scene: robot, cameras, joints, ROS2 topics           |
| 4    | [`04_physics_fixes.md`](04_physics_fixes.md)       | Physics issues we solved and how                                |
| 5    | [`05_vr_setup.md`](05_vr_setup.md)                 | ALVR + SteamVR + Quest 3 setup                                  |
| 6    | [`06_vr_teleoperation.md`](06_vr_teleoperation.md) | Running the VR teleoperation with Isaac Lab                     |
| 7    | [`07_troubleshooting.md`](07_troubleshooting.md)   | All known issues in one place                                   |
|      |                                                    |                                                                 |

---

## Day 1 Checklist

Everything is already set up on the workstation. You do **not** need to reinstall anything. These steps get you oriented and verify that the simulation side is working for you.

- [ ] Read `01_project_overview.md` and `02_isaac_sim_basics.md`
- [ ] Launch Isaac Sim and open the scene file: `~/mobile_ai_scene.usd`
- [ ] Press ▶ Play — robot should stand on the ground without sinking
- [ ] In a separate terminal, verify ROS2 topics are publishing:
  ```bash
  source /opt/ros/humble/setup.bash
  export ROS_USE_SIM_TIME=true
  ros2 topic list | grep -E "cam|joint|clock"
  ros2 topic hz /cam_high/color/image_raw   # expect ~36 Hz
  ros2 topic echo /joint_states --once      # expect 26 joints
  ```
- [ ] Read `05_vr_setup.md` and `06_vr_teleoperation.md`
- [ ] Run a VR session using the quick-start checklist in `05_vr_setup.md`
- [ ] Validate hand tracking with the Franka task (see `06_vr_teleoperation.md` Step 1)

---

## Environment at a Glance

| Component | Version / Location |
|---|---|
| OS | Ubuntu 22.04 |
| NVIDIA Driver | 580.x |
| Isaac Sim | 5.1.0 |
| Isaac Lab | 2.3.2 (`~/IsaacLab/`) |
| Trossen AI Isaac Extension | latest (`~/trossen_ai_isaac/`) |
| ROS2 | Humble |
| Scene file | `~/mobile_ai_scene.usd` |
| VR headset | Meta Quest 3 |
| VR stack | ALVR + SteamVR (OpenXR runtime) |
