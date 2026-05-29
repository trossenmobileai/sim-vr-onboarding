# 05 — VR Setup

This document covers the one-time configuration and the per-session startup procedure for VR teleoperation using the Meta Quest 3.

> **Everything in Part 1 (one-time setup) is already done on the workstation.** You can skip directly to the quick-start checklist at the bottom for your first session.

---

## Stack Overview

```
Meta Quest 3 (hand tracking + VR display)
        ↕  Wi-Fi (ALVR wireless stream)
ALVR Streamer  ──►  SteamVR  ──►  OpenXR API
                                        ↕
                              Isaac Sim / Isaac Lab
                              └── Renders stereo frames → SteamVR → ALVR → Quest 3 display
                              └── OpenXRDevice reads hand tracking → robot arm IK
```

**Why this stack?**

ALVR is the chosen approach because it requires no Docker or cloud infrastructure and can be set up in hours. A more integrated path (NVIDIA CloudXR) exists and may be adopted in a later phase. See the separate comparison document for details.

---

## Part 1 — One-Time System Configuration (Already Done)

These steps were performed once and persist across reboots. They are documented here for reference and in case the workstation is ever re-imaged.

### Step 1.1 — SteamVR Linux Capability Fix

SteamVR requires `CAP_SYS_NICE` to set high scheduling priority. Without it, SteamVR crashes after a few seconds.

```bash
sudo setcap CAP_SYS_NICE+eip \
  /home/trossen-admin/.steam/debian-installation/steamapps/common/SteamVR/bin/linux64/vrcompositor-launcher
```

Verify:

```bash
getcap /home/trossen-admin/.steam/debian-installation/steamapps/common/SteamVR/bin/linux64/vrcompositor-launcher
# Expected: ...vrcompositor-launcher cap_sys_nice=eip
```

### Step 1.2 — SteamVR Launch Option

Set in Steam → Library → right-click SteamVR → Properties → General → Launch Options:

```
/home/trossen-admin/.steam/debian-installation/steamapps/common/SteamVR/bin/vrmonitor.sh %command%
```

### Step 1.3 — ALVR Driver Registration

Created the file `~/.local/share/Steam/config/steamvr.vrsettings`:

```json
{
   "Driver_alvr_server" : {
      "enable" : true,
      "loadPriority" : 0
   },
   "steamvr" : {
      "activateMultipleDrivers" : true
   }
}
```

### Step 1.4 — SteamVR Set as OpenXR Runtime

Isaac Lab reads the system OpenXR runtime. SteamVR was registered as that runtime via:
SteamVR window → ☰ Menu → Settings → Developer → **Set SteamVR as OpenXR Runtime**

Verify:

```bash
cat ~/.config/openxr/1/active_runtime.json
# Must show "name": "SteamVR"
```

---

## Part 2 — Starting a VR Session (Every Session)

> ⚠️ **Order matters.** Follow these steps exactly — skipping or reordering will break the session.

### Step 2.1 — Launch ALVR

Open the **ALVR Launcher** on the workstation → click **Launch** next to the installed ALVR version.

### Step 2.2 — Launch SteamVR Through ALVR

Inside the ALVR dashboard → click **"Launch SteamVR"**.

> ⚠️ **Critical:** Always launch SteamVR from within ALVR — never directly from Steam. Launching SteamVR independently causes it to close after a few seconds because the ALVR driver is not loaded first.

### Step 2.3 — Connect the Meta Quest 3

1. On the Quest 3: App Library → **ALVR** → launch it
2. On the workstation: ALVR dashboard → **Devices** tab → Quest 3 appears → click **Trust**
3. The Quest 3 display switches from the Meta home to the **SteamVR home environment** ✅

### Step 2.4 — Configure Hand Tracking (Quest 3)

On the Quest 3 headset:
- Settings → Interaction → **Hand Tracking → ON**
- Settings → Interaction → **Auto Switch between Hands & Controllers → ON**

On the workstation ALVR dashboard:
- Settings → **Hand Tracking interaction → SteamVR Input 2.0**

---

## Quick-Start Checklist (Every Session)

Use this at the start of every VR session:

- [ ] Open ALVR Launcher → click **Launch**
- [ ] In ALVR dashboard → click **"Launch SteamVR"**
- [ ] On Quest 3 → open ALVR app
- [ ] ALVR Devices tab → Quest 3 trusted ✅
- [ ] Quest 3 shows SteamVR home ✅
- [ ] Open Isaac Sim → load `~/mobile_ai_scene.usd`
- [ ] **Rendering → VR → OpenXR → System OpenXR Runtime → Start VR** ✅
- [ ] Quest 3 shows Isaac Sim scene ✅
- [ ] Run teleop script (see `06_vr_teleoperation.md`)
- [ ] Press **S** to begin arm tracking ✅

---

## Key File Paths

| File | Path |
|------|------|
| SteamVR compositor | `/home/trossen-admin/.steam/debian-installation/steamapps/common/SteamVR/bin/linux64/vrcompositor-launcher` |
| SteamVR settings | `/home/trossen-admin/.local/share/Steam/config/steamvr.vrsettings` |
| Active OpenXR runtime | `~/.config/openxr/1/active_runtime.json` |
| Isaac Lab root | `~/IsaacLab/` |
| Trossen AI Isaac extension | `~/trossen_ai_isaac/` |
| NVIDIA modprobe config | `/etc/modprobe.d/nvidia.conf` |

---

## Reference Links

| Resource | URL |
|---|---|
| ALVR GitHub | https://github.com/alvr-org/ALVR |
| ALVR Installation Wiki | https://github.com/alvr-org/ALVR/wiki/Installation-guide |
| Isaac Lab Teleoperation docs | https://isaac-sim.github.io/IsaacLab/main/source/overview/imitation-learning/teleop_imitation.html |
| Trossen AI Isaac Extension | https://docs.trossenrobotics.com/trossen_arm/main/tutorials/trossen_ai_isaac.html |
| Reference video (Quest 2 + Isaac Lab) | https://www.youtube.com/watch?v=NUyL9z4tXUk |
