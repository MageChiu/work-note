# 04 · 自动扩缩容 Autoscaler

[← 上一篇：03 控制面](./03-control-plane.md) ｜ [返回首页](./README.md) ｜ 下一篇：[05 数据面 Router](./05-data-plane-router.md)

---

自动扩缩容逻辑全部在 [pkg/autoscaler](../../pkg/autoscaler)，是一个 **"指标采集 → 推荐 → 修正 → patch CR"** 的闭环。它作为控制面的一个控制器被启用（见 [03](./03-control-plane.md)）。

## 1. 总体流程

```
AutoscaleController.Reconcile()  （周期遍历 AutoscalingPolicyBinding）
        │
        ├─ 同构目标(HomogeneousTarget)  → doScale    → Autoscaler.Scale()
        └─ 异构目标(HeterogeneousTarget) → doOptimize → Optimizer
        │
        ▼
updateTargetReplicas（merge / json patch）
  → ModelServing.Spec.Replicas 或 role.replicas
```

## 2. 关键模块

### 2.1 控制器 — [controller/autoscale_controller.go](../../pkg/autoscaler/controller/autoscale_controller.go)
- 维护 `scalerMap` / `optimizerMap`，按 binding key 缓存实例。
- 创建/销毁 Autoscaler 与 Optimizer 实例，承担 reconcile 节拍。

### 2.2 指标采集 — [autoscaler/metric_collector.go](../../pkg/autoscaler/autoscaler/metric_collector.go)
- 分组拉取 `PodMetricSource`（按 `uri/port/selector` 复用一次抓取多指标）+ `PrometheusMetricSource`。
- 用 `expfmt` 解析 counter / gauge / histogram。
- histogram 通过 `QuantileInDiff` + 历史快照求差分位数；用 `SnapshotSlidingWindow` 维护 `PastHistograms`。

### 2.3 同构扩缩 — [autoscaler/scaler.go](../../pkg/autoscaler/autoscaler/scaler.go)
`Autoscaler.Scale()` 三步：
1. 采集指标
2. `RecommendedInstancesAlgorithm` 计算推荐副本
3. `CorrectedInstancesAlgorithm` 做窗口/Panic 修正后返回新副本数

### 2.4 异构优化 — [autoscaler/optimizer.go](../../pkg/autoscaler/autoscaler/optimizer.go)
- 多 backend 按 cost 排序，构建 `ScalingOrder ReplicaBlock`（支持 `costExpansionRatePercent` 分包）。
- 算出总实例数后 `RestoreReplicasOfEachBackend` 反推到每个 backend。

### 2.5 状态 — [autoscaler/status.go](../../pkg/autoscaler/autoscaler/status.go)
`Status` 含 PanicMode 时间戳 + 5 个滑动窗口：
`MaxRecommendation` / `MinRecommendation` / `MaxCorrected` / `MinCorrectedForStable` / `MinCorrectedForPanic`。

## 3. 核心算法

### 3.1 推荐算法 — [algorithm/recommendation.go](../../pkg/autoscaler/algorithm/recommendation.go)
HPA-like：
- tolerance 容差内不变更
- 外部指标按 `metric/target` 比例缩放
- 实例指标考虑 unready / missing 边界
- 多指标取 **max**

### 3.2 修正算法 — [algorithm/revision.go](../../pkg/autoscaler/algorithm/revision.go)
`CorrectedInstancesAlgorithm`：
- **Panic 模式**：快速扩容应对突发
- **Stable 模式**：区分 ScaleUp / ScaleDown，应用 absolute / percent 约束
- `SelectPolicy` 支持 AND / OR 组合
- 最终 clamp 到 `[Min, Max]`

## 4. 支撑数据结构

### 4.1 直方图 — [histogram/histogram.go](../../pkg/autoscaler/histogram/histogram.go)
`Snapshot{sum, count, buckets}` + `QuantileInDiff(percentile, now, past)`：基于**差分桶**估算窗口内分位数（避免被历史累积值污染）。

### 4.2 滑动窗口 — [datastructure/sliding_window.go](../../pkg/autoscaler/datastructure/sliding_window.go)
三种泛型滑动窗口：
- `RmqRecordSlidingWindow`：单调队列 RMQ，O(1) 取窗口极值
- `RmqLineChartSlidingWindow`：带 drift 的折线 RMQ
- `SnapshotSlidingWindow`：保留过期但仍可读的快照（供 histogram 差分使用）

## 5. 一句话总结

> 扩缩容 = "采集多源指标 → HPA-like 推荐副本 → 窗口/上下界/Panic 修正 → patch CR"；Optimizer 是同一 pipeline 面向多 backend、带成本权重的扩展。

相关示例见 [examples/keda-autoscaling](../../examples/keda-autoscaling) 与 [examples/prometheus-autoscaler](../../examples/prometheus-autoscaler)。

---

[← 上一篇：03 控制面](./03-control-plane.md) ｜ [返回首页](./README.md) ｜ 下一篇：[05 数据面 Router](./05-data-plane-router.md)
