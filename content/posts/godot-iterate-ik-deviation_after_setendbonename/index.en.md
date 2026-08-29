---
date: '2026-08-30T00:20:00+08:00'
draft: false
title: 'Godot IterateIK IK Deviation After SetEndBoneName'
tags: ["Godot", "C#"]
---

## Problem Description

Environment: Windows 11 + Godot 4.7 dotnet. Bones are driven by IK only (no animation). Reproduction steps:

1. Cut off the last bone of a chain (`SetEndBoneName` to shorten it)
2. Restore the remaining bones to a cached pose via `Skeleton.SetBonePose`
3. Move the marker to the target position of the new chain end
4. Call `MakeSimulationDirty()` to trigger a rebuild

> **Note**: `MakeSimulationDirty(index)` is a custom interface we added to `IterateIK3D` (the C# binding of `make_simulation_dirty()` in `iterate_ik_3d.h`), used to manually mark a single chain for rebuild. It is equivalent to the internal `_make_simulation_dirty()` — Godot does not expose it natively.

Observed behavior:

- **With joint constraints** (`rotation_axis` + `limitation` cone + `rotation_offset`): the IK result **deviates from the marker**, then **converges back after a few seconds**
- **Without constraints**: the IK result follows the marker directly, no issue at all

What made it odd: after restoring the cached pose, the chain end should already be on the marker (the IK "shouldn't move") — yet it first deviates, then slowly comes back.

## Investigation

The suspicion was constraints. Looking at `scene/resources/3d/joint_limitation_3d.cpp`, the cone check relies on `make_space()`:

```cpp
Quaternion JointLimitation3D::make_space(const Vector3 &p_local_forward_vector,
        const Vector3 &p_local_right_vector, const Quaternion &p_rotation_offset) const {
    Vector3 axis_y = p_local_forward_vector.normalized();
    // ...
    return (Quaternion(Vector3(0, 1, 0), axis_y) * p_rotation_offset.normalized()).normalized();
    // ↑ rotation_offset shifts the cone center here
}
```

`rotation_offset` shifts the cone center away from the bone's forward direction. Three experiments were run:

**Experiment 1: set the limitation cone angle to 360°** (effectively disabling the cone) — the cut is perfectly smooth, no deviation.

**Experiment 2: keep the cone angle, remove `rotation_offset`** — also fine.

**Experiment 3: keep both** — deviation reproduces.

Clear conclusion: **`rotation_offset` shifting the cone center is the culprit**. Right after a rebuild (the first frame after `MakeSimulationDirty`), the IK's judgment of the cone center is inconsistent with normal iteration, so the constraint misjudges and pulls the already-converged chain away; it then takes many frames to recover ("converges back after a few seconds"). Without constraints there is no cone check, hence no issue.

(The deeper mechanism: the rebuild initializes `lpose = lrest`, leading to `grest == gpose`, so the cone check degrades to a raw comparison of "original segment direction vs shifted cone center". Not expanded here.)

## Fix

Engine-side changes were risky, so we fixed it **game-side with fast convergence**: compress the "misjudgement → pull-away → gradual recovery" into a single frame — temporarily raise `max_iterations` and `angular_delta_limit` when cutting the chain, let the IK converge quickly, then restore the original values in `SkeletonUpdated`.

```csharp
// When cutting (in _UpdateIKConfig, before MakeSimulationDirty):
_SetIterateIKFastConverged(iterateIK);

// Fast-convergence toggle:
private void _SetIterateIKFastConverged(IterateIK3D ik)
{
    if (_iterateIKFastConverged) return;
    _iterateIKFastConverged = true;
    _iterateIKTempMaxIterations = ik.MaxIterations;
    _iterateIKTempAngularDeltaLimit = ik.AngularDeltaLimit;
    ik.MaxIterations = 9;                                    // default usually 4
    ik.AngularDeltaLimit = Mathf.DegToRad(10);               // default usually 2°
}

private void _UnsetIterateIKFastConverged(IterateIK3D ik)
{
    _iterateIKFastConverged = false;
    ik.MaxIterations = _iterateIKTempMaxIterations;
    ik.AngularDeltaLimit = _iterateIKTempAngularDeltaLimit;
}

// Restore in SkeletonUpdated (after IK is applied):
private void _OnSkeletonUpdated()
{
    if (_iterateIKFastConverged)
    {
        _UnsetIterateIKFastConverged(IK as IterateIK3D);
    }
    if (!IK.Active) return;
    // ...cache bone poses...
}
```

**Why it works**: the fast parameters provide `9 iterations × 10° ≈ 90°/frame` of convergence power (normal `4 × 2° = 8°/frame`, about 11× faster), so the post-rebuild misjudgement converges within one frame and the bones return to a normal state; after restoring the slow parameters, iteration stays smooth with no gradual recovery needed. Verified in practice.

**Notes**:

- If the deviation angle is close to or exceeds 90°, one frame may not be enough — instead, restore when `GetIKMarkerDistanceToChainEnd(chainIndex) < threshold`
- During the fast frame the bones may swing quickly (90° in one frame); acceptable visually, lower the delta if needed
- With multiple chains, the restore check must iterate over all chains
