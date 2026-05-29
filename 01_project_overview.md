# 01 — Project Overview

## The Big Picture

The goal of this project is to teach the **Trossen Mobile AI Platform** (a dual-arm mobile robot) to perform manipulation tasks — like picking up a cube — using **Imitation Learning (IL)**. The robot learns by watching human demonstrations, not by being explicitly programmed.

The full pipeline looks like this:

```
Real Robot
  └── Human teleoperates via leader arms
  └── Dataset recorded (camera images + joint states) via rosbag / LeRobot
          │
          ▼
Fine-tune Pi0.5 VLA model (via OpenPI)
          │
          ▼
Deploy fine-tuned model back to real robot
          │
          ▼
Robot performs task autonomously
```

---

## Where Simulation and VR Fit In

The sim/VR work runs in **parallel** to the real-robot pipeline and serves two purposes:

### 1. Digital Twin (Isaac Sim)
A virtual replica of the real robot and its environment runs in Isaac Sim. This lets the team:
- Test and debug ROS2 code without touching the hardware
- Generate **synthetic training data** at scale (thousands of episodes in simulation instead of hours of physical teleoperation)
- Validate the ROS2 topic structure before connecting to the real robot

The sim publishes the **exact same ROS2 topics** as the real robot (`/joint_states`, `/cam_high/color/image_raw`, etc.), so rosbag recordings from sim can feed directly into the VLA training pipeline.

### 2. VR Teleoperation (Quest 3 + Isaac Lab)
Instead of using the physical leader arms to collect demonstrations, an operator can wear a **Meta Quest 3** headset, see the simulation in stereo VR, and use their **bare hands** to control the robot arms in simulation.

```
Meta Quest 3 hand tracking
        │
        ▼
ALVR → SteamVR → OpenXR API
        │
        ▼
Isaac Lab OpenXRDevice
        │
        ▼
Trossen arm IK → joint commands → Isaac Sim robot
```

This is useful for collecting synthetic demonstrations more ergonomically and for validating the control pipeline before doing it on the real robot.

---

## Current Task

The simulation scene contains:
- The Trossen Mobile AI robot (`mobile_ai_scene.usd`)
- A table (represented as a cube)
- A red cube on the table (the manipulation target)

The current task being developed: **pick up the red cube** with one arm.

---

## Key Technologies

| Tool | Role |
|---|---|
| **Isaac Sim 5.1.0** | Physics simulator, Digital Twin environment |
| **Isaac Lab 2.3.2** | Robot learning framework, teleoperation scripts |
| **ROS2 Humble** | Communication layer (same as real robot) |
| **Pi0.5 / OpenPI** | VLA model being fine-tuned |
| **ALVR + SteamVR** | Wireless VR streaming to Meta Quest 3 |
| **Meta Quest 3** | VR headset + hand tracking device |

---

## Further Reading

- [NVIDIA Isaac Sim Documentation](https://docs.isaacsim.omniverse.nvidia.com/)
- [Isaac Lab Documentation](https://isaac-sim.github.io/IsaacLab/main/index.html)
- [Trossen AI Isaac Extension](https://docs.trossenrobotics.com/trossen_arm/main/tutorials/trossen_ai_isaac.html)
- [OpenPI (Pi0.5 fine-tuning)](https://github.com/Physical-Intelligence/openpi)
