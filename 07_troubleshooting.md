# 07 — Troubleshooting

All known issues from the simulation and VR setup, collected in one place.

---

## Simulation Issues

### S1 — Wheels Sink Through the Ground (Permanent)

**Cause:** Triangle mesh colliders are not supported on dynamic bodies in PhysX.
**Fix:** Change wheel collision mesh approximation to Convex Hull.
See [`04_physics_fixes.md`](04_physics_fixes.md) — Problem 1.

### S2 — Wheels Sink Intermittently

**Cause:** GPU broadphase buffer overflow — PhysX drops collision pairs silently.
**Fix:** Set `foundLostAggregatePairsCapacity` to 4096 on `/World/PhysicsScene`.
See [`04_physics_fixes.md`](04_physics_fixes.md) — Problem 2.

### S3 — `RuntimeError: Accessed invalid null prim` on Play/Stop

**Cause:** UI bug in `omni.physx.supportui` manipulator. No physics impact.
**Fix:** Deselect all prims before pressing Play or Stop.

```python
import omni.usd
omni.usd.get_context().get_selection().clear_selected_prim_paths()
```

### S4 — Depth Image Appears Black in RViz2

**Cause:** Depth values are not normalized for display by default.
**Fix:** In RViz2 Image display settings → enable **"Normalize Range"**.

### S5 — `sec: 0` on `/clock` Initially

**Cause:** `ros2_publish_clock` node's `timeStamp` input not connected.
**Fix:** In the Action Graph, connect `on_playback_tick → Time` to `ros2_publish_clock → timeStamp`.

### S6 — `/joint_states` Shows Wrong `targetPrim` Error

**Cause:** The articulation root prim path is wrong.
**Fix:** Use `/World/mobile_ai/base_footprint`, not `/World/mobile_ai`.

### S7 — Arms Drift Slowly Even With Zero Input

**Symptom:** Both robot arms drift slowly in simulation even when no teleop input is given and the teleop script is confirmed to be sending all-zero actions.  
**Cause:** Simulator-level numerical artifact in PhysX constraint solving for this robot model — not a bug in the teleop script. Persists with gravity disabled and high PD gains.  
**Full investigation:** See [`isaac_lab_teleop/03_arm_drift.md`](isaac_lab_teleop/03_arm_drift.md).  
**Workaround:** Accept the small drift. Design teleop sessions to be short episodes. Increasing PD gains (stiffness 800, damping 160) may reduce it.

---

## VR / ALVR Issues

### T1 — `Failed to set capabilities` When Running `setcap`

**Symptom:** `Failed to set capabilities on file '...' (No such file or directory)`
**Cause:** The path to `vrcompositor-launcher` is wrong.
**Fix:**

```bash
find / -name "vrcompositor-launcher" 2>/dev/null
# Use the returned path in the setcap command
```

### T2 — SteamVR Closes After a Few Seconds

**Cause:** Launch option not set, OR SteamVR was launched directly from Steam.
**Fix:**
1. Confirm launch option is set: Steam → SteamVR → Properties → Launch Options
2. Close SteamVR → relaunch through ALVR's "Launch SteamVR" button

### T3 — ALVR Warning: `steamvr.vrsettings does not exist`

**Symptom:** `Failed to unblock ALVR driver: .../steamvr.vrsettings does not exist`
**Cause:** ALVR driver registration file is missing.
**Fix:** Create it manually:

```bash
mkdir -p ~/.local/share/Steam/config
cat > ~/.local/share/Steam/config/steamvr.vrsettings << 'EOF'
{
   "Driver_alvr_server" : { "enable" : true, "loadPriority" : 0 },
   "steamvr" : { "activateMultipleDrivers" : true }
}
EOF
```

Then restart ALVR and SteamVR.

### T4 — Quest 3 Not Appearing in ALVR Devices Tab

**Cause:** Network issue — devices on different networks, or ALVR not running on Quest 3.
**Fix:**
- Confirm both devices are on the same **5 GHz Wi-Fi** network
- On Quest 3: App Library → launch ALVR app
- If on a university/institutional network: use a **dedicated router** — institutional networks often block peer-to-peer UDP traffic required by ALVR

### T5 — Quest 3 Shows Black Screen After Connecting

**Cause:** SteamVR compositor running but not rendering to headset.
**Fix:**
1. ALVR Settings → Video → reduce **Encode Resolution** (e.g., 100% → 75%)
2. Restart ALVR + SteamVR session
3. ALVR Settings → Controllers → Hand Tracking → confirm set to **SteamVR Input 2.0**

### T6 — Isaac Sim Crashes on "Start VR" (Segmentation Fault)

**Symptom:** `vrclient.so!CGpuTiming::GetDeltas` → `Segmentation fault (core dumped)`
**Cause:** Conflict between NVIDIA driver 580.x and SteamVR's GPU frame timing compositor on Linux.

**Fix — try in order:**

**Option A (most effective) — Disable NVIDIA GPU Firmware:**

```bash
sudo nano /etc/modprobe.d/nvidia.conf
# Add: options nvidia NVreg_EnableGpuFirmware=0
sudo update-initramfs -u
sudo reboot
```

**Option B — Disable Motion Smoothing:**

SteamVR → ☰ → Settings → Video → **Motion Smoothing → Off** → restart VR session

**Option C — Switch SteamVR to Beta:**

Steam → Library → right-click SteamVR → Properties → Betas → **"beta - SteamVR Beta Update"**

### T7 — `XR_ERROR_RUNTIME_UNAVAILABLE` When Running Isaac Lab Script

**Cause:** No active OpenXR runtime — SteamVR not running or not set as OpenXR runtime.
**Fix:**
1. Confirm ALVR and SteamVR are running and Quest 3 is connected (see `05_vr_setup.md` Part 2)
2. Verify OpenXR runtime:

```bash
cat ~/.config/openxr/1/active_runtime.json
# Must show "name": "SteamVR"
```

3. If wrong runtime: SteamVR → Settings → Developer → **Set SteamVR as OpenXR Runtime**

### T8 — Repeated `Desync detected. Attempting recovery` in ALVR Log

**Cause:** Insufficient Wi-Fi signal quality or interference.
**Fix:**
- Move Quest 3 closer to the router
- Switch to **5 GHz** band if on 2.4 GHz
- Use a **dedicated router** rather than a shared network
- ALVR Settings → Video → reduce **Encode Bitrate** (e.g., 200 Mbps → 100 Mbps)

### T9 — Hand Tracking Works but Arm Doesn't Move

**Cause:** Arm tracking is paused (default state on script launch).
**Fix:** Press **S** on the keyboard to start arm tracking.
