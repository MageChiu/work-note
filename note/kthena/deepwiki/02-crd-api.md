# 02 · CRD 与 API 模型

[← 上一篇：01 架构总览](./01-architecture-overview.md) ｜ [返回首页](./README.md) ｜ 下一篇：[03 控制面](./03-control-plane.md)

---

CRD 定义全部位于 [pkg/apis](../../pkg/apis)，分为两个 API Group：

- `workload.serving.volcano.sh/v1alpha1` —— 工作负载与扩缩容（[pkg/apis/workload/v1alpha1](../../pkg/apis/workload/v1alpha1)）
- `networking.serving.volcano.sh/v1alpha1` —— 流量路由（[pkg/apis/networking/v1alpha1](../../pkg/apis/networking/v1alpha1)）

## 1. CRD 全景与关系链

```
        ModelBooster  (高阶"一键发模型")
              │ 渲染
   ┌──────────┼───────────────┐
   ▼          ▼               ▼
ModelRoute  ModelServer   ModelServing
(对外模型名  (后端 Pod 池   (核心工作负载)
 → 后端映射)  声明)              │ 渲染
                                ▼
                          ServingGroup  (Gang 调度单位)
                                │ 创建
                                ▼
                              Pods (多角色)

AutoscalingPolicy + AutoscalingPolicyBinding  → 绑定到 ModelServing/后端，驱动扩缩容
```

> 数据面（Router）只读取 **ModelRoute / ModelServer / Pod** 三类对象。

## 2. workload 组

### 2.1 ModelBooster — [model_booster_types.go](../../pkg/apis/workload/v1alpha1/model_booster_types.go)
"一键发模型"高阶 CRD，封装 **模型获取 + 后端运行时 + 自动扩缩容 + 自动生成 ModelRoute**。

关键字段：
- `Spec.Owner`
- `Spec.Backend`（`ModelBackend`）：
  - `Type`、`ModelURI`（`hf://` / `s3://` / `pvc://` / `ms://`）、`CacheURI`
  - `MinReplicas` / `MaxReplicas`
  - `Workers []ModelWorker`，每个 worker 的 `Type ∈ {server, prefill, decode, controller, coordinator}`，含 `Replicas`、`Pods`、`Resources`、`Affinity`、引擎参数透传
  - `SchedulerName`、`RuntimeClassName`
- `Spec.AutoscalingPolicy`
- `Spec.ModelMatch`：用于自动生成 ModelRoute 的匹配规则

### 2.2 ModelServing — [model_serving_types.go](../../pkg/apis/workload/v1alpha1/model_serving_types.go)
核心工作负载 CRD，类似 Deployment + StatefulSet 的混合体，承载一个或多个（多角色）Pod 模板。

关键字段：
- `Spec`：`Replicas`、`SchedulerName`、`Plugins []PluginSpec(scope)`、`Template`、`RolloutStrategy`（`ServingGroupRollingUpdate` / `RoleRollingUpdate`）、`RecoveryPolicy`
- `Status`：`Replicas`、`CurrentRevision`、`UpdateRevision`、`Conditions`

### 2.3 ServingGroup — [servinggroup_types.go](../../pkg/apis/workload/v1alpha1/servinggroup_types.go)
一个 Pod 副本组，是 **Gang 调度** 的最小单位；一个 ModelServing 副本通常对应一个 ServingGroup。

关键字段：
- `GangPolicy`：原子调度策略
- `NetworkTopology`：Volcano 网络拓扑感知调度
- `Roles []Role`：每个 Role 含 `EntryTemplate`，可选 `WorkerReplicas + WorkerTemplate`（Leader-Worker 模式）

### 2.4 AutoscalingPolicy / AutoscalingPolicyBinding
定义扩缩容策略（多指标、上下界、cost-aware 异构目标）与绑定目标，详见 [04 · 自动扩缩容](./04-autoscaler.md)。

### 2.5 标签约定 — [labels.go](../../pkg/apis/workload/v1alpha1/labels.go)
统一标签 key（控制器选 Pod、判断滚动、识别 PD 角色都靠它们）：
```
modelserving.volcano.sh/name
modelserving.volcano.sh/group-name
modelserving.volcano.sh/role
modelserving.volcano.sh/role-id
modelserving.volcano.sh/entry
modelserving.volcano.sh/revision
modelserving.volcano.sh/role-template-hash
```

## 3. networking 组

### 3.1 ModelRoute — [modelroute_types.go](../../pkg/apis/networking/v1alpha1/modelroute_types.go)
把对外模型名（如 `gpt-4o`）映射到一组 ModelServer 后端，支持 LoRA、加权分流、限流。是 Router 路由匹配的**最高优先级**。

关键 struct：
- `ModelRouteSpec`
  - `ModelName`：对外暴露的模型名
  - `LoraAdapters []string`：可选 LoRA 列表
  - `ParentRefs`：Gateway API 风格父引用
  - `Rules []Rule`：匹配 + 加权目标
  - `RateLimit *RateLimit`：input/output token 限流，时间单位、可选 Redis 全局
- `Rule { ModelMatch, TargetModels []TargetModel(weight) }`
- `ModelMatch { Headers / Uri / Body []StringMatch }`，支持 `Exact` / `Prefix` / `Regex`

### 3.2 ModelServer — [modelserver_types.go](../../pkg/apis/networking/v1alpha1/modelserver_types.go)
声明一组同构后端 Pod 集合（一个具体模型/引擎），Router 通过它选 Pod。

关键 struct：
- `ModelServerSpec`
  - `Model`：对引擎呈现的模型名（可重写）
  - `InferenceEngine`：`vLLM` / `SGLang` / `MindIE` 等
  - `WorkloadSelector { MatchLabels, PDGroup *PDGroup }`：选 Pod；PD 模式下用 `PrefillLabels` / `DecodeLabels` / `GroupKey` 区分角色
  - `WorkloadPort`：业务端口
  - `TrafficPolicy { Timeout, Retry }`
  - `KVConnector` 类型：`http` / `nixl` / `lmcache` / `mooncake`

## 4. 生成物与 codegen

CRD 的 deepcopy、defaulter、clientset、informers、listers、applyconfiguration 全部由 codegen 生成：
- 触发：`make generate` → [hack/update-codegen.sh](../../hack/update-codegen.sh)
- CRD YAML：`make gen-crd`（controller-gen）输出到各 chart 的 `crds/` 目录，详见 [06 · CLI 与部署制品](./06-cli-deploy.md)
- 生成的客户端在 [client-go/](../../client-go)，**禁止手编辑**

---

[← 上一篇：01 架构总览](./01-architecture-overview.md) ｜ [返回首页](./README.md) ｜ 下一篇：[03 控制面](./03-control-plane.md)
