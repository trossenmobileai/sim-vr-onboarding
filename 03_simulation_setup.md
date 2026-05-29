# 03 — Simulation Setup

This document describes the Isaac Sim scene that is already built and saved in `~/mobile_ai_scene.usd`. You do not need to rebuild it — this is a reference so you understand what's inside.

---

## Scene Contents

The scene (`/World`) contains:

- **`/World/mobile_ai`** — the Trossen Mobile AI robot (imported from URDF → USD)
- **`/World/Table`** — a static cube representing the table
- **`/World/RedCube`** — a red cube placed on the table (the manipulation target)
- **`/World/PhysicsScene`** — global PhysX settings
- **`/World/ActionGraph`** — ROS2 publisher graph (see below)

---

## Stage Hierarchy: Robot Prims

### Articulation Root

```
/World/mobile_ai/base_footprint   [ArticulationRoot]
```
This is the entry point PhysX uses to control the entire robot. Always reference this path when publishing joint states or sending joint commands.

### Cameras (3× Intel RealSense D405)

| Camera | USD Path | Mount |
|--------|----------|-------|
| `cam_high` | `/World/mobile_ai/cam_high_link/cam_high_color_frame/Camera_high` | Top of mobile base |
| `cam_left` | `/World/mobile_ai/follower_left_camera_link/follower_left_camera_color_frame/Camera_follower_left` | Left arm wrist |
| `cam_right` | `/World/mobile_ai/follower_right_camera_link/follower_right_camera_color_frame/Camera_follower_right` | Right arm wrist |

Each camera link contains the full RealSense frame structure:

```
*_color_frame → *_color_optical_frame + Camera_* [Camera prim]
*_depth_frame → *_depth_optical_frame
*_infra1_frame → *_infra1_optical_frame
*_infra2_frame → *_infra2_optical_frame
```

### Robot Joints (26 total)

| Group | Joint Names | Count | Type |
|-------|-------------|-------|------|
| Left arm | `follower_left_joint_0` → `follower_left_joint_5` | 6 | RevoluteJoint |
| Right arm | `follower_right_joint_0` → `follower_right_joint_5` | 6 | RevoluteJoint |
| Left gripper | `follower_left_left_carriage_joint`, `follower_left_right_carriage_joint` | 2 | PrismaticJoint |
| Right gripper | `follower_right_left_carriage_joint`, `follower_right_right_carriage_joint` | 2 | PrismaticJoint |
| Drive wheels | `left_wheel`, `right_wheel` | 2 | RevoluteJoint |
| Caster swivels | `caster_swivel_joint_front_left/right`, `caster_swivel_joint_rear_left/right` | 4 | RevoluteJoint |
| Caster wheels | `caster_wheel_joint_front_left/right`, `caster_wheel_joint_rear_left/right` | 4 | RevoluteJoint |

---

## Action Graph: ROS2 Publisher

The Action Graph at `/World/ActionGraph` runs every simulation tick (60 Hz) and publishes the following:

```
[on_playback_tick] (60 Hz)
    │
    ├──► Camera High  → render product (640×480)
    │         ├──► /cam_high/color/image_raw       [rgb,   ~36 Hz]
    │         └──► /cam_high/depth/image_rect_raw  [depth, ~36 Hz]
    │
    ├──► Camera Left  → render product (640×480)
    │         ├──► /cam_left/color/image_raw
    │         └──► /cam_left/depth/image_rect_raw
    │
    ├──► Camera Right → render product (640×480)
    │         ├──► /cam_right/color/image_raw
    │         └──► /cam_right/depth/image_rect_raw
    │
    ├──► /joint_states    [sensor_msgs/JointState, 60 Hz, 26 joints]
    └──► /clock           [rosgraph_msgs/Clock,    60 Hz]
```

---

## ROS2 Topics Reference

| Topic | Message Type | Rate | Notes |
|-------|-------------|------|-------|
| `/cam_high/color/image_raw` | `sensor_msgs/Image` | ~36 Hz | RGB, 640×480, `rgb8` |
| `/cam_high/depth/image_rect_raw` | `sensor_msgs/Image` | ~36 Hz | Depth, 640×480, `32FC1` |
| `/cam_left/color/image_raw` | `sensor_msgs/Image` | ~36 Hz | RGB, 640×480, `rgb8` |
| `/cam_left/depth/image_rect_raw` | `sensor_msgs/Image` | ~36 Hz | Depth, 640×480, `32FC1` |
| `/cam_right/color/image_raw` | `sensor_msgs/Image` | ~36 Hz | RGB, 640×480, `rgb8` |
| `/cam_right/depth/image_rect_raw` | `sensor_msgs/Image` | ~36 Hz | Depth, 640×480, `32FC1` |
| `/joint_states` | `sensor_msgs/JointState` | ~60 Hz | 26 joints, sim-time stamped |
| `/clock` | `rosgraph_msgs/Clock` | ~60 Hz | Simulation clock |

---

## Environment Configuration

Add to `~/.bashrc` (already done on the workstation):

```bash
source /opt/ros/humble/setup.bash
export ROS_USE_SIM_TIME=true
```

> ⚠️ Always launch RViz2 and rosbag in a **separate terminal** from Isaac Sim to avoid `libOgreMain.so` library conflicts.

---

## Verification Commands

```bash
# List all relevant topics
ros2 topic list | grep -E "cam|joint|clock"

# Check camera publish rate (~36 Hz expected)
ros2 topic hz /cam_high/color/image_raw

# Verify joint states (26 joints, with sim timestamp)
ros2 topic echo /joint_states --once

# Verify simulation clock
ros2 topic echo /clock --once

# Visualize camera feeds (run in a clean separate terminal)
rviz2
# In RViz2: Add → By topic → /cam_high/color/image_raw → Image
# For depth: enable "Normalize Range" in display settings
```

---

## Key Files

| File | Description |
|------|-------------|
| `~/mobile_ai_scene.usd` | Main Isaac Sim scene (robot + Action Graph + environment) |
| `~/.bashrc` | Contains `ROS_USE_SIM_TIME=true` and ROS2 source |
