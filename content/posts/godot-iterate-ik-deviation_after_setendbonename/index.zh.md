---
date: '2026-08-30T00:20:00+08:00'
draft: false
title: 'Godot IterateIK 调用 SetEndBoneName 后的 IK 偏离问题'
tags: ["Godot", "C#"]
---

## 问题描述

本机环境为 Windows11 + Godot 4.7 dotnet，骨骼只由 IK 驱动（无动画）。复现步骤：

1. 切断链条的最后一个 bone（`SetEndBoneName` 缩短链条）
2. 用 `Skeleton.SetBonePose` 把剩余骨骼摆到缓存姿态
3. 把 marker 移到新链条末端的目标位置
4. 调用 `MakeSimulationDirty()` 触发重建

> **注**：`MakeSimulationDirty(index)` 是我们在 `IterateIK3D` 上添加的自定义接口（`iterate_ik_3d.h` 中 `make_simulation_dirty()` 的 C# 绑定），用于单条链手动标记重建，等价于引擎内部 `_make_simulation_dirty()`——Godot 原生未公开。

观察到的现象：

- **有关节约束**（`rotation_axis` + `limitation` cone + `rotation_offset`）：IK 结果**偏离 marker**，但**几秒后重新重合**
- **无约束**：IK 结果直接贴合 marker，无任何异常

更奇怪的是：恢复缓存姿态后链条末端理论上已经在 marker 上（IK 不应该动），但它先偏了，然后慢慢回来。

## 深入调查

首先怀疑与约束有关。查阅 `scene/resources/3d/joint_limitation_3d.cpp`，锥约束的判定依赖 `make_space()`：

```cpp
Quaternion JointLimitation3D::make_space(const Vector3 &p_local_forward_vector,
        const Vector3 &p_local_right_vector, const Quaternion &p_rotation_offset) const {
    Vector3 axis_y = p_local_forward_vector.normalized();
    // ...
    return (Quaternion(Vector3(0, 1, 0), axis_y) * p_rotation_offset.normalized()).normalized();
    // ↑ rotation_offset 在这里偏移锥中心
}
```

`rotation_offset` 会把锥中心从骨骼 forward 方向偏移出去。于是做了三组实验：

**实验 1：把 limitation 锥角调到 360°**（相当于禁用锥限制）——断开瞬间完全正常，无偏离。

**实验 2：保留锥角、去掉 `rotation_offset`**——也正常。

**实验 3：两者都保留**——偏离复现。

结论清晰了：**`rotation_offset` 偏移锥中心是元凶**。IK **重建瞬间**（`MakeSimulationDirty` 后第一帧）对锥中心的判断与正常迭代不一致，导致约束误判，把已收敛的链条掰偏，之后需要多帧渐进修复（"几秒后重合"）。无约束时没有锥检查，自然没有这个问题。

（注：深层机制上，这是重建初始化 `lpose = lrest` 导致 `grest == gpose`、锥检查退化为"原始段方向 vs 偏移锥中心"的裸判所致——此处不展开。）

## 修复

既然引擎侧改动风险大，改为**游戏侧快速收敛**：把"误判 → 掰偏 → 渐进修复"的收敛过程压缩到 1 帧内完成——切断时临时调大 `max_iterations` 和 `angular_delta_limit`，让 IK 快速收敛，随后在 `SkeletonUpdated` 恢复原参数。

```csharp
// 切断时（_UpdateIKConfig 里，MakeSimulationDirty 之前）：
_SetIterateIKFastConverged(iterateIK);

// 快速收敛开关：
private void _SetIterateIKFastConverged(IterateIK3D ik)
{
    if (_iterateIKFastConverged) return;
    _iterateIKFastConverged = true;
    _iterateIKTempMaxIterations = ik.MaxIterations;
    _iterateIKTempAngularDeltaLimit = ik.AngularDeltaLimit;
    ik.MaxIterations = 9;                                    // 正常通常 4
    ik.AngularDeltaLimit = Mathf.DegToRad(10);               // 正常通常 2°
}

private void _UnsetIterateIKFastConverged(IterateIK3D ik)
{
    _iterateIKFastConverged = false;
    ik.MaxIterations = _iterateIKTempMaxIterations;
    ik.AngularDeltaLimit = _iterateIKTempAngularDeltaLimit;
}

// SkeletonUpdated（IK 应用后）恢复：
private void _OnSkeletonUpdated()
{
    if (_iterateIKFastConverged)
    {
        _UnsetIterateIKFastConverged(IK as IterateIK3D);
    }
    if (!IK.Active) return;
    // ...缓存骨骼姿态...
}
```

**为什么有效**：fast 参数提供 `9 次迭代 × 10° ≈ 90°/帧` 的收敛能力（正常 `4 × 2° = 8°/帧`，快约 11 倍），1 帧内即可把重建瞬间的误判收敛完，骨骼回到正常状态；恢复慢速后平滑维持，不再需要渐进修复。实测通过。

**注意事项**：

- 若偏差角接近/超过 90°，1 帧可能不够——可改为在 `GetIKMarkerDistanceToChainEnd(chainIndex) < 阈值` 时再恢复
- fast 帧骨骼会快速甩动（1 帧转 90°），视觉可接受，必要时降低 delta
- 多个 chain 时，恢复判断需遍历所有 chain
