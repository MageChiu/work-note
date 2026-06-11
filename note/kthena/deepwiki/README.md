# Kthena DeepWiki

> 本目录是 **Kthena** 代码仓库的本地 DeepWiki 文档，参考 [deepwiki.com](https://deepwiki.com/volcano-sh/kthena) 的组织方式，把整个代码库拆解为多篇互相链接的 Wiki 页面，便于快速理解架构与定位源码。

Kthena 是一个 **Kubernetes 原生的 LLM 推理平台**（Go module: `github.com/volcano-sh/kthena`），通过一套 CRD + 控制器 + 智能路由器，把 vLLM / SGLang / Triton / MindIE 等多种推理引擎统一编排，提供声明式模型生命周期管理、PD（Prefill-Decode）解耦、成本驱动自动扩缩容与智能流量管理。

## 文档导航

| 页面 | 内容 |
|------|------|
| [01 · 架构总览](./01-architecture-overview.md) | 整体定位、组件拓扑、控制面/数据面划分、端到端数据流 |
| [02 · CRD 与 API 模型](./02-crd-api.md) | `pkg/apis` 下 6 个 CRD 的字段、Status、关系链 |
| [03 · 控制面 Controller Manager](./03-control-plane.md) | `pkg/controller` 控制器注册、Webhook、Leader Election |
| [04 · 自动扩缩容 Autoscaler](./04-autoscaler.md) | 指标采集 → 推荐 → 修正 → patch 的闭环算法 |
| [05 · 数据面 Router](./05-data-plane-router.md) | 路由、调度插件、datastore、KV Connectors、过滤器 |
| [06 · CLI 与部署制品](./06-cli-deploy.md) | CLI、Helm Charts、client-go、Docker 镜像 |
| [07 · 工程化基建](./07-engineering.md) | Makefile、CI workflows、codegen、e2e、Python 组件 |
| [08 · 术语表](./08-glossary.md) | 关键名词与缩写速查 |

## 快速画像

```
            ┌──────────────────  Kubernetes API  ──────────────────┐
            │  CRD (pkg/apis)                                      │
            │   workload.serving.volcano.sh/                       │
            │     ModelBooster · ModelServing · ServingGroup ·     │
            │     AutoscalingPolicy / AutoscalingPolicyBinding     │
            │   networking.serving.volcano.sh/                     │
            │     ModelRoute · ModelServer                         │
            └───────────────┬───────────────────────┬───────────────┘
                            │                       │
       kthena-controller-manager           kthena-router
       （控制面：reconcile + webhook        （数据面：OpenAI 兼容网关
        + autoscaler）                       + 调度 + PD 路由）
                            │                       │
                            └──────► Pods (vLLM / SGLang / MindIE) ◄──────┘
```

## 如何使用本 Wiki

- 第一次了解项目：按 `01 → 02 → 03/04 → 05` 顺序阅读。
- 排障/二开数据面：直接看 [05 · 数据面 Router](./05-data-plane-router.md)。
- 部署/运维：看 [06 · CLI 与部署制品](./06-cli-deploy.md) 与 [07 · 工程化基建](./07-engineering.md)。
- 文档中所有源码引用均给出了相对仓库根目录的路径，可直接在 IDE 中跳转。

> 说明：本 Wiki 基于对仓库源码的静态阅读整理而成，与官方 [docs/kthena](../../docs/kthena/) 文档互补——官方文档面向使用者，本 Wiki 面向**读代码、改代码**的工程师。
