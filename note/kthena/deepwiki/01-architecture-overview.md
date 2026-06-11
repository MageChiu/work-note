# 01 · 架构总览

[← 返回首页](./README.md) ｜ 下一篇：[02 · CRD 与 API 模型](./02-crd-api.md)

---

## 1. 项目定位

Kthena 是一个 **Kubernetes 原生的 LLM 推理平台**，核心目标是用云原生的声明式方式管理大模型推理工作负载。它的关键能力：

- **声明式模型生命周期**：发布、滚动更新、扩缩容全部通过 CRD 声明，控制器负责 reconcile。
- **PD 解耦（Prefill–Decode disaggregation）**：控制面用多角色 Pod 编排 prefill/decode，数据面用 KV Connector 串联请求。
- **成本驱动自动扩缩容**：多指标 + 异构后端 + 滑动窗口 + Panic/Stable 双模式。
- **智能流量管理**：模型名/Header/URI 路由、限流、公平调度、prefix-cache 感知、KV-Cache 感知、LoRA 感知。
- **多引擎支持**：vLLM、SGLang、Triton、MindIE。

## 2. 两大可独立部署的平面

| 平面 | 二进制 | 入口 | 源码主体 |
|------|--------|------|----------|
| 控制面 | `kthena-controller-manager` | [cmd/kthena-controller-manager/main.go](../../cmd/kthena-controller-manager/main.go) | `pkg/controller`、`pkg/autoscaler`、`pkg/apis` |
| 数据面 | `kthena-router` | [cmd/kthena-router/main.go](../../cmd/kthena-router/main.go) | `pkg/kthena-router` |

两条平面**可单独部署**，控制面还可通过 `--controllers` 开关裁剪具体控制器。

## 3. 组件拓扑

```
            ┌─────────────────────────  Kubernetes API  ───────────────────────────┐
            │  CRD (pkg/apis):                                                      │
            │   ├─ workload.serving.volcano.sh/{ModelBooster, ModelServing,         │
            │   │      ServingGroup, AutoscalingPolicy, AutoscalingPolicyBinding}   │
            │   └─ networking.serving.volcano.sh/{ModelRoute, ModelServer}          │
            └────────────────────┬─────────────────────────────┬────────────────────┘
                                 │                             │
            kthena-controller-manager                 kthena-router
            ┌──────────────────────────────┐         ┌────────────────────────────────┐
            │ pkg/controller (开关入口)      │         │ pkg/kthena-router              │
            │  ├ ModelBooster controller    │         │  ├ controller (Gateway/Route   │
            │  ├ ModelServing controller    │         │  │   /InferencePool informers) │
            │  │   (+ 可选 LWS 集成)         │         │  ├ datastore (Store + 公平队列  │
            │  └ Autoscaler controller     │         │  │   + token tracker + 飞行计数 │
            │     (pkg/autoscaler)          │         │  │   + Redis 选项)             │
            │                                │         │  ├ router (HTTP 入口/路由匹配) │
            │ Webhook server :8443           │         │  ├ scheduler (插件化打分/过滤) │
            │ (validating + mutating)        │         │  ├ filters (auth/ratelimit/   │
            │ Debug HTTP (localhost only)    │         │  │   tokenizer)               │
            └──────────────────────────────┘         │  ├ backend (vLLM/SGLang 指标) │
                                                       │  ├ connectors (HTTP/NIXL/      │
                                                       │  │   Mooncake/SGLang KV)      │
                                                       │  ├ accesslog/metrics/debug   │
                                                       │  └ Webhook server :8443       │
                                                       └────────────────────────────────┘
                                                                    │
                                                  ┌─────────── 数据面流量 ───────────┐
                                                  │ vLLM / SGLang / MindIE Pods     │
                                                  └─────────────────────────────────┘
```

## 4. 端到端数据流

### 4.1 部署 / 发布流
```
用户 apply CRD
  → ModelBooster 渲染出 ModelRoute + ModelServer + ModelServing
  → ModelServing 控制器渲染出 ServingGroup（Gang 单位）
  → ServingGroup 创建 Pods（多角色：server / prefill / decode / coordinator）
  → kthena-router 的 informers 把 ModelRoute/ModelServer/Pod 同步进 Store
```

### 4.2 请求处理流（数据面）
```
HTTP /v1/chat/completions
  → AccessLogMiddleware (request_id + 计时)
  → JWTAuthenticator (Bearer，写 UserId)
  → Router.HandlerFunc
      ├ Tokenizer.CalculateTokenNum
      ├ TokenRateLimiter（input token 限流，可选 Redis 全局）
      ├ FairnessQueue.PushRequest（按 user+model 优先级排队）
      ├ doLoadbalance：ModelRoute 优先 > HTTPRoute(InferencePool)
      ├ Store 取候选 Pod / PDGroupPods
      ├ SchedulerImpl
      │     ├ Filter (lora-affinity, least-request 上限)
      │     ├ Score  (gpu / least-latency / least-request / prefix / kvcache-aware / random)
      │     └ PD 模式：先选 decode topN，再为每个 decode 选 prefill
      ├ proxyModelEndpoint
      │     ├ 普通：transport 反向代理
      │     └ PD：connectors.KVConnector.Proxy(prefillAddr, decodeAddr, hooks)
      └ 解析 usage → TokenTracker.UpdateTokenCount → output 限流 → Prometheus → access log
```

### 4.3 扩缩容闭环（控制面）
```
Pod metrics（vLLM/SGLang 指标 + Prometheus）
  → Autoscaler.Scale / Optimizer
      → RecommendedInstancesAlgorithm（HPA-like，多指标取 max）
      → CorrectedInstancesAlgorithm（Stable/Panic 双模式 + 上下界 clamp）
  → patch ModelServing.Spec.Replicas / role.replicas
  → ModelServing 控制器调整 Pod 数 → 回到数据面
```

## 5. 目录速览

| 目录 | 角色 |
|------|------|
| [pkg/apis](../../pkg/apis) | CRD API 定义（networking / workload 两组 v1alpha1） |
| [pkg/controller](../../pkg/controller) | 控制器注册入口与配置 |
| [pkg/autoscaler](../../pkg/autoscaler) | 自动扩缩容算法与控制器 |
| [pkg/kthena-router](../../pkg/kthena-router) | 数据面 Router 全部逻辑 |
| [cmd/](../../cmd) | 两个二进制入口（薄壳） |
| [cli/kthena](../../cli/kthena) | `kthena` CLI 与内嵌模板 |
| [charts/kthena](../../charts/kthena) | Helm Chart（父 + networking/workload 子 chart） |
| [client-go/](../../client-go) | codegen 生成的 K8s 客户端 |
| [python/](../../python) | downloader / runtime 两个 Python 组件 |
| [docker/](../../docker) | Dockerfile 集合 |
| [test/e2e/](../../test/e2e) | 基于 Kind 的端到端测试 |
| [hack/](../../hack) | codegen、版权、本地部署脚本 |

## 6. 关键设计亮点

1. **CRD 分层**：`ModelBooster`（一键发布）→ `ModelServing/ServingGroup`（多角色编排）→ `ModelRoute/ModelServer`（流量层）。
2. **PD 解耦贯通两个平面**：控制面用角色 + PDGroup 标签表达，数据面用 PDGroupPods + KVConnector + 两阶段调度执行。
3. **可插拔调度框架**：filter / score 插件 + ConfigMap 配置，亮点是 prefix-cache 与 KV-cache 感知。
4. **优雅降级**：Redis 不可达自动回退本地限流/计数；webhook 证书三优先级取材；Gateway API 可选启用。

---

[← 返回首页](./README.md) ｜ 下一篇：[02 · CRD 与 API 模型](./02-crd-api.md)
