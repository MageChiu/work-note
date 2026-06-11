# 08 · 术语表

[← 上一篇：07 工程化基建](./07-engineering.md) ｜ [返回首页](./README.md)

---

| 术语 / 缩写 | 含义 |
|-------------|------|
| **CRD** | Custom Resource Definition，Kubernetes 自定义资源。Kthena 用它声明模型工作负载与路由，见 [02](./02-crd-api.md) |
| **Control Plane / 控制面** | `kthena-controller-manager`，负责 reconcile CRD、webhook、扩缩容，见 [03](./03-control-plane.md) |
| **Data Plane / 数据面** | `kthena-router`，承接推理流量的网关与调度器，见 [05](./05-data-plane-router.md) |
| **ModelBooster** | 高阶"一键发模型"CRD，封装模型获取 + 运行时 + 扩缩容 + 路由生成 |
| **ModelServing** | 核心工作负载 CRD，多角色 Pod 模板，类似 Deployment+StatefulSet |
| **ServingGroup** | Pod 副本组，Gang 调度的最小单位 |
| **ModelRoute** | 把对外模型名映射到后端的路由 CRD（最高优先级路由匹配） |
| **ModelServer** | 一组同构后端 Pod 池的声明，Router 据此选 Pod |
| **PD / Prefill-Decode 解耦** | 把"prefill（编码 prompt）"与"decode（逐 token 生成）"拆到不同 Pod，优化硬件利用率 |
| **PDGroup** | ModelServer 中用 `PrefillLabels`/`DecodeLabels`/`GroupKey` 表达的 PD 分组 |
| **KV Connector** | PD 模式下串联 prefill 与 decode 的 KV cache 传输实现（http/nixl/mooncake/sglang），见 [05 §10](./05-data-plane-router.md) |
| **NIXL** | 一种 KV cache 远程传输方式：prefill 返回 `kv_transfer_params`，decode 携回拉取 |
| **Mooncake** | KV 传输方案，代码层复用 NIXL 实现 |
| **LMCache** | KV 缓存方案，代码层复用 HTTP connector |
| **vLLM / SGLang / MindIE / Triton** | 支持的推理引擎；Router 为前两者实现了指标抓取 |
| **Tokenizer** | 计算请求 token 数的组件，用于限流；有 estimator 与 tiktoken 两种实现 |
| **Fairness Queue / 公平队列** | 按 user+model 优先级排队的请求队列，防止单用户挤占 |
| **TokenTracker** | 按 (user,model) 滑窗统计 token 用量，input/output 加权（默认 1:2） |
| **OnFlight Counter / 飞行计数** | 统计每 Pod 在途请求数，可用 Redis 做跨副本视图 |
| **Prefix Cache 感知** | 调度插件，用 xxHash 分块哈希 prompt 前缀，命中越多的 Pod 越优先 |
| **KV-Cache 感知** | 调度插件，根据后端 kv-cache 命中度评分 |
| **LoRA** | 低秩适配器，可热插拔；`lora-affinity` 过滤器据此选 Pod |
| **Gang Scheduling** | 整组原子调度，避免分布式推理组部分部署浪费资源 |
| **Network Topology 感知** | 把实例放在同一网络域以提升互联带宽 |
| **Autoscaler Panic / Stable 模式** | 扩缩容两种节奏：Panic 快速扩容应对突发，Stable 受限速约束，见 [04](./04-autoscaler.md) |
| **AutoscalingPolicy / Binding** | 扩缩容策略 CRD 与其目标绑定 |
| **Optimizer** | 面向多 backend、带成本权重的扩缩容优化器 |
| **Volcano** | 批调度器，Kthena 集成其 Gang/网络拓扑调度能力 |
| **Gateway API / InferencePool** | Kubernetes Gateway API 及其推理扩展，Router 可选支持 |
| **LWS / LeaderWorkerSet** | 一种 Leader-Worker 工作负载，ModelServing 可选集成 |
| **client-go** | codegen 生成的 K8s 客户端，禁止手编辑，见 [06 §3](./06-cli-deploy.md) |
| **codegen** | `make generate` 触发的代码生成（deepcopy/clientset/informers/CRD/docs） |

---

[← 上一篇：07 工程化基建](./07-engineering.md) ｜ [返回首页](./README.md)
