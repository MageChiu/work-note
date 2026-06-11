# 06 · CLI 与部署制品

[← 上一篇：05 数据面 Router](./05-data-plane-router.md) ｜ [返回首页](./README.md) ｜ 下一篇：[07 工程化基建](./07-engineering.md)

---

本页覆盖 `kthena` CLI、Helm Charts、client-go、Docker 镜像等"用户可见与可部署"的制品。

## 1. CLI — [cli/kthena](../../cli/kthena)

### 1.1 入口与命令树
- [main.go](../../cli/kthena/main.go)：用 `//go:embed helm/templates/**/*.yaml` 把模板编译进二进制，`cmd.InitTemplates(templatesFS)` 后 `cmd.Execute()`。
- [cmd/root.go](../../cli/kthena/cmd/root.go)：cobra 根命令 `kthena`，并提供 `GetRootCmd()` 给 docgen 复用。
- [cmd/create.go](../../cli/kthena/cmd/create.go)：`create manifest`——加载 values（YAML + `--set`）→ Helm engine 渲染 → 打印 YAML → 确认 → apply 到集群。参数 `--template/-t`、`--values-file/-f`、`--dry-run`、`-n`、`--name`、`--set`。
- [cmd/get.go](../../cli/kthena/cmd/get.go)：`get templates` / `get template <NAME>` / `get model-boosters` / `get model-servings` / `get autoscaling-policies`（支持 `-n` / `--all-namespaces`）。
- [cmd/describe.go](../../cli/kthena/cmd/describe.go)：`describe template/model-booster/model-serving/autoscaling-policy`。
- [cmd/templates.go](../../cli/kthena/cmd/templates.go)：模板访问器（`InitTemplates`、`GetTemplateContent`、`ListTemplates`、`GetTemplateInfo`）；模板路径结构 `helm/templates/<vendor>/<model>.yaml`，从注释解析 `# Description:`。

### 1.2 内置模型模板 — [cli/kthena/helm/templates](../../cli/kthena/helm/templates)
- `Qwen/`：Qwen3-8B / 32B（含 `-NPU`）、Qwen3-Coder-30B-A3B-Instruct（含 NPU 与 PD-Disaggregated 的 LMCache / Mooncake / Nixl 三种变体）。
- `deepseek-ai/`：DeepSeek-R1-Distill-Qwen-7B / 32B（含 `-NPU`）。
- 模板刻意保持 quick-start 最简化，不收录复杂方案。

### 1.3 文档生成 — [internal/tools/docgen/main.go](../../cli/kthena/internal/tools/docgen/main.go)
独立程序，调 `cmd.GetRootCmd()` + cobra `doc.GenMarkdownTree`，把 CLI 帮助生成到 `docs/kthena/docs/reference/kthena-cli/`；由 `make gen-docs` 触发。

### 1.4 kubectl 插件 — [cli/kubectl-plugin](../../cli/kubectl-plugin)
当前仅 `.gitkeep` 占位，**无功能实现**。

## 2. Helm Charts — [charts/kthena](../../charts/kthena)

### 2.1 父 + 子 chart 结构
- 父 chart [Chart.yaml](../../charts/kthena/Chart.yaml) `kthena`，依赖两个子 chart：
  - **networking**（条件 `networking.enabled`）：CRDs（modelroutes/modelservers）+ kthena-router Deployment/Service/ConfigMap/Webhook/RBAC。
  - **workload**（条件 `workload.enabled`）：CRDs（modelboosters/modelservings/autoscalingpolicies/policybindings）+ controller-manager Deployment + RBAC + 双 webhook + secret。
- 父 chart 自身只有 [templates/_helpers.tpl](../../charts/kthena/templates/_helpers.tpl)（命名工具）。

### 2.2 关键配置
- [values.yaml](../../charts/kthena/values.yaml)：
  - `workload.controllerManager.image / debugPort / webhook / downloaderImage / runtimeImage`
  - `networking.kthenaRouter.{port=8080, debugPort=15000, image, tls, webhook, fairness, gatewayAPI, terminationGracePeriodSeconds=330, drainTimeout=5m}`
  - `global.certManagementMode ∈ {auto(默认自签), cert-manager, manual}`；`global.webhook.caBundle` 仅 `manual` 用
- [values.schema.json](../../charts/kthena/values.schema.json)：仅强约束 `global.certManagementMode` 枚举与 `caBundle`，其它放开。

### 2.3 与部署的关系
- CRD 直接放在子 chart 的 `crds/` 目录（由 `make gen-crd` 用 controller-gen 输出），`helm install` 自动安装。
- router Deployment 的 `args` 直接拼接 `--port` / `--debug-port` / `--enable-webhook` / `--enable-gateway-api[-inference-extension]` / `--webhook-*` / `--kube-api-qps/burst`，并通过 `POD_NAMESPACE` / `REDIS_*` env 注入 fairness 与 redis 配置。

## 3. client-go — [client-go/](../../client-go)
**完全是 codegen 产物**，由 [hack/update-codegen.sh](../../hack/update-codegen.sh) 调 `kube_codegen.sh` 生成（`make generate` 触发）：
- `clientset/versioned/`：含 `typed/networking/v1alpha1/{modelroute,modelserver}`、`typed/workload/v1alpha1/{autoscalingpolicy,autoscalingpolicybinding,modelbooster,modelserving}`，均带 `fake/`。
- `informers/externalversions/` 与 `listers/`：覆盖上述全部类型。
- `applyconfiguration/`：覆盖所有 CRD 类型与嵌套类型。
- AGENTS.md 明确：`client-go/` **不允许手编辑**。

## 4. Docker 镜像 — [docker/](../../docker)
- [Dockerfile.kthena-controller-manager](../../docker/Dockerfile.kthena-controller-manager)：`golang` 多阶段 → distroless，镜像 `ghcr.io/volcano-sh/kthena-controller-manager`。
- [Dockerfile.kthena-router](../../docker/Dockerfile.kthena-router)：同上，镜像 `ghcr.io/volcano-sh/kthena-router`。
- [Dockerfile.mooncake-npu-a3](../../docker/Dockerfile.mooncake-npu-a3)：基于 Ascend vLLM 构建带 Mooncake KV 传输的 NPU/A3 运行镜像。
- `downloader` / `runtime` 镜像的 Dockerfile 在 [python/Dockerfile](../../python/Dockerfile)（多 target），不在 `docker/`。

## 5. 一键本地部署
[hack/local-up-kthena.sh](../../hack/local-up-kthena.sh)：本地 Kind/集群一键部署（`make docker-build-all` + helm 安装到 `kthena-system`），README 中 `./hack/local-up-kthena.sh` 即此脚本。

---

[← 上一篇：05 数据面 Router](./05-data-plane-router.md) ｜ [返回首页](./README.md) ｜ 下一篇：[07 工程化基建](./07-engineering.md)
