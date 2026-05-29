# 04 — Physics Fixes

When the robot is first simulated, the wheels sink through the ground. This document explains why it happens, how it was fixed, and how to verify the fix is applied. The fix is already applied in `~/mobile_ai_scene.usd` — this is a reference in case the scene is re-imported from URDF.

---

## Problem 1: Triangle Mesh Colliders on Wheels

### Why It Happens

When a URDF robot is imported into Isaac Sim, collision meshes default to **triangle mesh** colliders. PhysX only supports triangle mesh colliders on **static objects** (walls, floors). On dynamic/moving bodies like wheels, PhysX silently ignores them — the wheels exist visually but have no collision, so they fall through the ground.

### Affected Prims

Every wheel collision mesh follows this pattern:
```
/World/mobile_ai/<link_name>/collisions/<link_name>/node_STL_BINARY_/mesh  [Mesh]
```

Specifically for the drive and caster wheels:
```
/World/mobile_ai/left_wheel_link/collisions/wheel/node_STL_BINARY_/mesh
/World/mobile_ai/right_wheel_link/collisions/wheel/node_STL_BINARY_/mesh
/World/mobile_ai/caster_wheel_front_left/collisions/caster_wheel/node_STL_BINARY_/mesh
/World/mobile_ai/caster_wheel_front_right/collisions/caster_wheel/node_STL_BINARY_/mesh
/World/mobile_ai/caster_wheel_rear_left/collisions/caster_wheel/node_STL_BINARY_/mesh
/World/mobile_ai/caster_wheel_rear_right/collisions/caster_wheel/node_STL_BINARY_/mesh
```

### Fix: Change Approximation to Convex Hull

**Option A — Manual (Isaac Sim UI):**
1. In the Stage panel, navigate to a wheel collision mesh prim (the `[Mesh]` node inside `/collisions/`)
2. In the Property panel → Physics → Collider, find the **Approximation** dropdown
3. Change `Triangle Mesh` → `Convex Hull`
4. Repeat for all 6 wheel prims
5. Stop ⏹ → Play ▶ to reload physics

**Option B — Script (Isaac Sim Script Editor: Window → Script Editor):**

```python
import omni.usd
from pxr import UsdPhysics, PhysxSchema

stage = omni.usd.get_context().get_stage()

wheel_collision_meshes = [
    "/World/mobile_ai/left_wheel_link/collisions/wheel/node_STL_BINARY_/mesh",
    "/World/mobile_ai/right_wheel_link/collisions/wheel/node_STL_BINARY_/mesh",
    "/World/mobile_ai/caster_wheel_front_left/collisions/caster_wheel/node_STL_BINARY_/mesh",
    "/World/mobile_ai/caster_wheel_front_right/collisions/caster_wheel/node_STL_BINARY_/mesh",
    "/World/mobile_ai/caster_wheel_rear_left/collisions/caster_wheel/node_STL_BINARY_/mesh",
    "/World/mobile_ai/caster_wheel_rear_right/collisions/caster_wheel/node_STL_BINARY_/mesh",
]

for path in wheel_collision_meshes:
    prim = stage.GetPrimAtPath(path)
    if not prim:
        print(f"NOT FOUND: {path}")
        continue
    UsdPhysics.CollisionAPI.Apply(prim)
    PhysxSchema.PhysxCollisionAPI.Apply(prim)
    mesh_api = UsdPhysics.MeshCollisionAPI.Apply(prim)
    mesh_api.GetApproximationAttr().Set("convexHull")
    print(f"Fixed: {path}")

print("Done! Stop and Play the simulation to apply changes.")
```

### Collider Type Reference

| Approximation | Dynamic Bodies | Notes |
|---|---|---|
| Triangle Mesh | ❌ Not supported | Static geometry only — silently ignored on moving links |
| **Convex Hull** | ✅ Recommended | Wraps the STL in one solid convex shell — reliable and efficient |
| Sphere | ✅ Good for round wheels | Most stable but may not fit wide/flat wheels |
| SDF Mesh | ✅ Most accurate | High GPU cost — avoid for multi-link robots |

---

## Problem 2: GPU Broadphase Buffer Overflow

### Why It Happens

PhysX GPU broadphase maintains a buffer of all potential collision pairs. The Trossen robot (two 6-DOF arms, 8 caster assemblies, base chassis) has many links, and the number of collision pairs exceeds the default buffer size. When the buffer overflows, PhysX logs an error and **silently drops collision pairs** — causing intermittent or partial sinking even when Convex Hull is set correctly.

The error in the Isaac Sim log looks like:

```
[Error] [omni.physx.plugin] PhysX error: The application needs to increase
PxGpuDynamicsMemoryConfig::foundLostAggregatePairsCapacity to 3459
```

### Fix: Increase GPU Memory Config

Run in the Script Editor:

```python
import omni.usd
from pxr import PhysxSchema

stage = omni.usd.get_context().get_stage()
scene = stage.GetPrimAtPath("/World/PhysicsScene")

physx_scene = PhysxSchema.PhysxSceneAPI.Apply(scene)
physx_scene.GetGpuFoundLostAggregatePairsCapacityAttr().Set(4096)
physx_scene.GetGpuMaxNumPartitionsAttr().Set(8)
physx_scene.GetGpuTotalAggregatePairsCapacityAttr().Set(1024 * 1024)

print("GPU memory config updated.")
```

> Set `foundLostAggregatePairsCapacity` to at least the value in the error message, rounded up to the next power of 2 (e.g., error says 3459 → set 4096).

---

## Problem 3: `RuntimeError: Accessed invalid null prim`

This error fires during Play/Stop transitions:
```
RuntimeError: Accessed invalid null prim
KeyError: <class 'NoneType'>
  File "rigid_body_transform_manipulator.py"
```

This is a **UI bug** in `omni.physx.supportui` — it does not affect physics simulation. It is cosmetic log noise.

**Fix:** Deselect all prims before pressing Play or Stop:

```python
import omni.usd
omni.usd.get_context().get_selection().clear_selected_prim_paths()
```

Or simply click on empty space in the Viewport before pressing ▶.

---

## Diagnostic: Verify Physics APIs on the Stage

To confirm the fixes are applied, run this in the Script Editor:

```python
import omni.usd
from pxr import UsdPhysics

stage = omni.usd.get_context().get_stage()

def print_hierarchy(prim, indent=0):
    prefix = "  " * indent
    tags = []
    if UsdPhysics.CollisionAPI(prim): tags.append("COLLIDER")
    if UsdPhysics.RigidBodyAPI(prim): tags.append("RIGIDBODY")
    if UsdPhysics.ArticulationRootAPI(prim): tags.append("ARTICULATION_ROOT")
    tag_str = f"  ← [{', '.join(tags)}]" if tags else ""
    print(f"{prefix}[{prim.GetTypeName()}] {prim.GetPath().name}{tag_str}")
    for child in prim.GetChildren():
        print_hierarchy(child, indent + 1)

print_hierarchy(stage.GetPrimAtPath("/World/mobile_ai"))
```

Expected output for a correctly fixed wheel:

```
[Xform] left_wheel_link  ← [RIGIDBODY]
  [Xform] collisions
    [Xform] wheel
      [Xform] node_STL_BINARY_
        [Mesh] mesh  ← [COLLIDER]   ✅
```

---

## Summary

| Problem | Root Cause | Fix |
|---------|-----------|-----|
| Wheels sink permanently | Triangle mesh colliders ignored on dynamic bodies | Change approximation to Convex Hull on all 6 wheel collision mesh prims |
| Wheels sink intermittently | GPU broadphase buffer too small | Set `foundLostAggregatePairsCapacity` to 4096 on PhysicsScene |
| Null prim error on Play/Stop | UI bug in `omni.physx.supportui` | Deselect all prims before Play/Stop — no physics impact |

> Both the collider fix and the GPU memory fix are required. The collider type is the structural root cause; the GPU memory overflow amplifies the problem under load.
