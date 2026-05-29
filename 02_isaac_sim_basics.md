# 02 — Isaac Sim Basics

> **For ROS2 users new to Isaac Sim.** This is a short conceptual primer. You do not need to master everything here before proceeding — read it once to build a mental model, then come back as needed.

---

## What Is Isaac Sim?

Isaac Sim is NVIDIA's robotics simulator built on the **Omniverse** platform. Think of it as a physics-accurate 3D world that:
- Simulates robot joints, contact forces, friction, and gravity (via **PhysX GPU**)
- Renders photorealistic camera images (ray-tracing)
- Publishes ROS2 topics just like a real robot would
- Can run Isaac Lab training environments in parallel

If you know Gazebo, Isaac Sim fills a similar role but with significantly higher visual and physics fidelity, and GPU-parallel environments.

---

## Key Concepts

### USD — Universal Scene Description

Isaac Sim uses **USD** (originally from Pixar, adopted by NVIDIA) as its scene format instead of SDF/XACRO. Think of a `.usd` file as the Isaac Sim equivalent of a Gazebo world file — it describes everything in the scene: robots, tables, lights, cameras, physics settings.

Every object in the scene is a **prim** (primitive) with a **path**, for example:
```
/World/mobile_ai          ← the robot root
/World/mobile_ai/cam_high_link/...  ← a camera link
/World/PhysicsScene       ← global physics settings
```

The tree of prims is called the **Stage**. You can browse it in Isaac Sim's **Stage panel** (similar to a ROS TF tree, but for the scene).

### Action Graph

An **Action Graph** is Isaac Sim's visual scripting system for connecting simulation events to outputs — in our case, connecting the simulation tick to ROS2 publishers. It is configured once and saved inside the `.usd` file.

Our Action Graph (at `/World/ActionGraph`) does three things every simulation tick:
1. Renders camera images and publishes them as ROS2 topics
2. Publishes joint states as `/joint_states`
3. Publishes the simulation clock as `/clock`

You do not need to edit the Action Graph unless you add new sensors or topics.

### PhysX

Isaac Sim uses **NVIDIA PhysX** as its physics engine (GPU-accelerated). Key things to know:
- **Articulation:** A robot with joints is modeled as a PhysX articulation — a chain of rigid bodies connected by joints.
- **Collision geometry:** Every link that should collide with other objects needs a collider. PhysX has rules about which collider types work on moving vs. static objects (see `04_physics_fixes.md`).
- **Physics Scene:** Global settings like gravity, GPU memory, and solver iterations live at `/World/PhysicsScene`.

### ROS2 Bridge

Isaac Sim has a built-in **ROS2 bridge extension** (`isaacsim.ros2.bridge`). When enabled, it allows Action Graph nodes to publish and subscribe to ROS2 topics directly — no separate bridge node needed. The simulation clock is published to `/clock`, and `ROS_USE_SIM_TIME=true` must be set in your terminal so ROS2 tools use sim time instead of wall time.

---

## Useful Resources

- [Isaac Sim Getting Started](https://docs.isaacsim.omniverse.nvidia.com/5.1.0/introductory_tutorials/tutorial_intro_overview.html)
- [USD Concepts Overview](https://docs.isaacsim.omniverse.nvidia.com/5.1.0/scene_authoring/usd_overview.html)
- [Action Graph Introduction](https://docs.isaacsim.omniverse.nvidia.com/5.1.0/omnigraph/omnigraph_overview.html)
- [Isaac Sim ROS2 Bridge](https://docs.isaacsim.omniverse.nvidia.com/5.1.0/ros2_tutorials/tutorial_ros2_overview.html)
- [YouTube: Getting Started with Isaac Lab](https://www.youtube.com/watch?v=oGczfNwR-Uw)
