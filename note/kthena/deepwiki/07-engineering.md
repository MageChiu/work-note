# 07 · 工程化基建

[← 上一篇：06 CLI 与部署制品](./06-cli-deploy.md) ｜ [返回首页](./README.md) ｜ 下一篇：[08 术语表](./08-glossary.md)

---

本页覆盖构建/测试/发布工具链、CI workflows、Python 组件、e2e 与 benchmark。

## 1. Makefile 关键目标

| 目标 | 作用 |
|------|------|
| `make gen-crd` | controller-gen 生成 networking/workload CRD 到各 chart `crds/` |
| `make generate` | `gen-crd` + controller-gen object(deepcopy) + `go mod tidy` + [update-codegen.sh](../../hack/update-codegen.sh) + `gen-docs` + `gen-copyright` |
| `make gen-docs` | crd-ref-docs + CLI docgen + helm-docs |
| `make gen-check` | `make generate` 后 `git diff --exit-code` 校验生成物是否同步 |
| `make fmt` / `vet` / `lint` / `lint-fix` | gofmt / go vet / golangci-lint |
| `make lint-python` | ruff（`python/kthena/pyproject.toml`） |
| `make test` | `make generate` + `go test`（排除 e2e、client-go）+ 覆盖率 |
| `make test-docs` | `cd docs/kthena && npm run typecheck && npm run build` |
| `make build` | 生成 `bin/{kthena-router, kthena-controller-manager, kthena}` |
| `make test-e2e-controller-manager / -router / -gateway-api / -gateway-inference-extension` | 分类 e2e |
| `make docker-build-{router,controller,downloader,runtime,all}` | 构建镜像 |
| `make docker-buildx` | 多架构构建并推送 |
| `make licenses-check` / `lint-licenses` | 许可证镜像与校验 |

## 2. GitHub Workflows — [.github/workflows](../../.github/workflows)

| Workflow | 触发 | 主要执行 |
|----------|------|----------|
| `go-tests.yml` | Go 源码变更 | `make test` + 覆盖率门槛 |
| `go-check.yml` | 非 docs/.github 变更 | `helm lint` + `make gen-check` + `make lint` |
| `e2e-tests.yml` | pkg/cmd/test/config/charts/docker/hack/Makefile 变更 | matrix 跑四类 `test-e2e-*`（kind-action），失败收集 `kind export logs`，最后 cleanup |
| `docs-tests.yml` | `docs/kthena/**` | `make test-docs` |
| `python-tests.yml` | `python/**` | pytest + coverage ≥60% |
| `python-lint.yml` | `python/**` | `make lint-python` |
| `licenses-lint.yaml` / `python-licenses-lint.yml` | 许可证检查 |
| `codespell.yaml` | 拼写检查 |
| `build-push-release.yml` | push main 或 tag `v*` | buildx 构建并推送四个镜像 + CLI 多平台二进制 + 改写 chart version/appVersion/image.tag + `helm push` 到 OCI + 生成 `kthena-install.yaml` + GitHub Release |
| `approve-workflow.yml` / `retest.yml` | bot 工作流 |

## 3. hack 脚本 — [hack/](../../hack)
- [update-codegen.sh](../../hack/update-codegen.sh)：codegen 入口（deepcopy + clientset/informers/listers/applyconfig）。
- [update-crd.sh](../../hack/update-crd.sh)：把 controller-gen 生成的 CRD YAML 同步到 chart `crds/`。
- [update-copyright.sh](../../hack/update-copyright.sh)：批量补版权头（`make gen-copyright`）。
- [licenses-check.sh](../../hack/licenses-check.sh)：许可证校验。
- [local-up-kthena.sh](../../hack/local-up-kthena.sh)：本地一键部署。

## 4. Python 组件 — [python/kthena](../../python/kthena)
- **downloader**（`python/kthena/downloader/`）：FastAPI/CLI 下载器，支持 `hf://` / `ms://` / `s3://` / `obs://` / `pvc://`，含并发下载与文件锁；对应镜像 target `downloader`。
- **runtime**（`python/kthena/runtime/`）：sidecar 风格 metrics 代理 + 模型运行辅助，标准化 vLLM/SGLang 指标，提供 LoRA 加载/卸载、模型下载 API；对应镜像 target `runtime`。
- 测试 `python/tests/`；构建 [python/Dockerfile](../../python/Dockerfile)（多 target）。

## 5. e2e 测试 — [test/e2e](../../test/e2e)
- `setup.sh` / `cleanup.sh`：基于 Kind 建/销集群（默认 `kthena-e2e`，K8s v1.31.0），按 `TEST_CATEGORY` 决定额外安装的 CRD（LeaderWorkerSet / Gateway API / Inference Extension）。
- `framework/`、`utils/`：测试基础设施（chat、config、lora、pod、portforward 等 helper）。
- 四类测试目录：`controller-manager/`、`router/`、`router/gateway-api/`、`router/gateway-inference-extension/`，与 Make 目标一一对应。

## 6. Benchmark — [benchmark/kthena-router](../../benchmark/kthena-router)
基于 sglang `bench_serving.py` 打包成 Docker 镜像，作为 Kubernetes Job 跑路由器/推理压测，支持 hostPath 缓存 tokenizer/数据。

## 7. Examples — [examples/](../../examples)
按场景分组的演示 YAML：
- `kthena-router/`：Gateway、HTTPRoute、InferencePool、ModelRoute*、ModelServer*、LLM-Mock* 等。
- `model-serving/`：gangPolicy、gpu-kvcache-aware、gpu-pd-disaggregation、multi-node、network-topology、role-rollingupdate、rollingupdate、sample。
- `model-booster/`、`models/`、`keda-autoscaling/`、`prometheus-autoscaler/`、`redis/`。

> AGENTS.md 要求：CRD shape 变化时同步更新 `examples/`。

---

[← 上一篇：06 CLI 与部署制品](./06-cli-deploy.md) ｜ [返回首页](./README.md) ｜ 下一篇：[08 术语表](./08-glossary.md)
