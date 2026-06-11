# 05 · 数据面 Router

[← 上一篇：04 自动扩缩容](./04-autoscaler.md) ｜ [返回首页](./README.md) ｜ 下一篇：[06 CLI 与部署制品](./06-cli-deploy.md)

---

数据面二进制 `kthena-router` 是承接 OpenAI 兼容请求的多租户网关 + 调度器。入口 [cmd/kthena-router/main.go](../../cmd/kthena-router/main.go) 与 [cmd/kthena-router/app](../../cmd/kthena-router/app)，业务主体在 [pkg/kthena-router](../../pkg/kthena-router)。

## 1. 二进制入口与启动时序

[cmd/kthena-router/main.go](../../cmd/kthena-router/main.go) + [app/server.go](../../cmd/kthena-router/app/server.go)：

### 命令行参数
- 数据面：`--port=8080`、`--tls-cert`、`--tls-key`、`--debug-port=15000`
- 特性开关：`--enable-gateway-api`、`--enable-gateway-api-inference-extension`（要求前者也开启）
- Webhook：`--enable-webhook`（默认 true）、`--webhook-port=8443`、`--webhook-tls-*`、`--cert-secret-name`、`--webhook-service-name`
- 客户端流控：`--kube-api-qps`、`--kube-api-burst`

### 启动时序
1. 异步 `runWebhook`：CA bundle 三优先级（Secret → 文件 → 自动签发），写回 `ValidatingWebhookConfiguration kthena-router-validating-webhook`，cert ready 后在 8443 起 TLS。
2. `app.NewServer(...).Run(ctx)`：
   - 按 `REDIS_HOST` 决定是否启用 Redis-backed 全局 on-flight 计数器（`datastore.WithRedisOnFlightCounter`）。
   - `datastore.New(...)` → `NewRouter(store, /etc/config/routerConfiguration.yaml)`（router 必须先于 controller 创建，因为要向 store 注册回调）。
   - [app/controller.go](../../cmd/kthena-router/app/controller.go) 启动 `ModelRouteController` / `ModelServerController`；启用 Gateway API 时确保默认 GatewayClass/Gateway 存在并启动 `GatewayController`；启用 Inference Extension 时再加 `HTTPRouteController` 与基于 dynamic client 的 `InferencePoolController`。
   - `WaitForCacheSync` 后 `store.Run(ctx)` 启动周期更新循环。
   - `startRouter`：localhost:15000 暴露 debug + pprof；Gateway API 模式用 `ListenerManager` 按 listener 动态启停子 server，否则单一 `:8080`，统一暴露 `/v1/*`、`/healthz`、`/readyz`、`/metrics`。
   - 关闭用 `DRAIN_TIMEOUT`（默认 5min）做 graceful drain。

## 2. 子模块全景

| 子目录 | 职责 |
|--------|------|
| [controller/](../../pkg/kthena-router/controller) | informer 控制器：同步 ModelRoute/ModelServer/Gateway/HTTPRoute/InferencePool 进 Store |
| [router/](../../pkg/kthena-router/router) | HTTP 入口主流程 + HTTPRoute 匹配 |
| [handlers/](../../pkg/kthena-router/handlers) | OpenAI 请求/响应解析（含流式 usage） |
| [filters/](../../pkg/kthena-router/filters) | auth（JWT）、ratelimit（token 限流）、tokenizer |
| [scheduler/](../../pkg/kthena-router/scheduler) | 插件化 filter/score 调度框架 |
| [datastore/](../../pkg/kthena-router/datastore) | 核心数据层 + 公平队列 + token tracker + 飞行计数 |
| [backend/](../../pkg/kthena-router/backend) | vLLM/SGLang 指标抓取与归一化 |
| [connectors/](../../pkg/kthena-router/connectors) | PD 解耦 KV Connector（HTTP/NIXL/Mooncake/SGLang） |
| [accesslog/](../../pkg/kthena-router/accesslog) ｜ [metrics/](../../pkg/kthena-router/metrics) ｜ [debug/](../../pkg/kthena-router/debug) | 访问日志 / Prometheus 指标 / 调试 dump |
| [common/](../../pkg/kthena-router/common) ｜ [utils/](../../pkg/kthena-router/utils) | 上下文键、prompt 解析、Redis 优雅降级 |

## 3. 控制器层 — [controller/](../../pkg/kthena-router/controller)
五个 informer 控制器把对应资源回调写入 `Store.AddOrUpdateXxx` / `DeleteXxx`：
- [gateway_controller.go](../../pkg/kthena-router/controller/gateway_controller.go)：仅处理 `GatewayClassName == "kthena-router"`，用 Workqueue + `initialSyncSignal` 在缓存就绪前阻断流量。
- [constants.go](../../pkg/kthena-router/controller/constants.go)：`DefaultGatewayClassName="kthena-router"`、`ControllerName="volcano.sh/kthena-router"`、`maxRetries=5`。

## 4. 路由主体 — [router/router.go](../../pkg/kthena-router/router/router.go)
`Router.HandlerFunc()` 主流程（最大文件，约 1000+ 行）：
1. 解析 OpenAI 请求体（model / prompt / messages）
2. **输入限流**（按 Tokenizer 计算的 input token）
3. **公平调度入队**（可选，按 user+model 优先级）
4. `doLoadbalance`：先匹 ModelRoute，其次 HTTPRoute（可绑 InferencePool）
5. `Scheduler` 选 Pod（PD 模式选 prefill+decode 对）
6. `proxyModelEndpoint`：普通模式直接反向代理；PD 分离调用 `connectors.KVConnector.Proxy(...)`
7. 解析 usage → token tracker → output token 限流 → metrics → access log

[httproute_match.go](../../pkg/kthena-router/router/httproute_match.go)：hostnames 支持 `*.x.com` 通配；path 优先级 `Exact > PathPrefix > Regex`。

## 5. 请求/响应解析 — [handlers/](../../pkg/kthena-router/handlers)
- [request.go](../../pkg/kthena-router/handlers/request.go)：`OpenAIRequestBody{Model}` 解析。
- [response.go](../../pkg/kthena-router/handlers/response.go)：`OpenAIResponse{Usage{prompt/completion/total}}`；`ParseStreamRespForUsage` 解析 SSE `data:` 流式 usage。

## 6. 过滤器 — [filters/](../../pkg/kthena-router/filters)
- **auth** [authentication.go](../../pkg/kthena-router/filters/auth/authentication.go)：`JWTAuthenticator` Gin 中间件，校验 Bearer token，`JWKSRotator` 自动轮换 JWKS，校验 iss/aud/exp/nbf/iat（1 分钟 skew），sub 写入 `common.UserIdKey`。
- **ratelimit** [ratelimit.go](../../pkg/kthena-router/filters/ratelimit/ratelimit.go)：每模型一对 input/output Limiter；区分 `InputRateLimitExceededError` / `OutputRateLimitExceededError`；[global.go](../../pkg/kthena-router/filters/ratelimit/global.go) 提供 Redis 全局滑窗，本地用 `x/time/rate`。
- **tokenizer** [tokenizer.go](../../pkg/kthena-router/filters/tokenizer/tokenizer.go)：`Tokenizer interface { CalculateTokenNum(string) (int, error) }`，两个实现 [estimator.go](../../pkg/kthena-router/filters/tokenizer/estimator.go)（启发式）与 [tiktoken.go](../../pkg/kthena-router/filters/tokenizer/tiktoken.go)（OpenAI BPE）。

## 7. 调度器 — [scheduler/](../../pkg/kthena-router/scheduler)
- 接口 [framework/interface.go](../../pkg/kthena-router/scheduler/framework/interface.go)：`Context{Model, Prompt, Hashes, ModelServerName, PDGroup, DecodePods, PrefillPods, BestPods, MetricsRecorder}`、`ScorePlugin` / `FilterPlugin` / `PostScheduleHook`。
- [scheduler_impl.go](../../pkg/kthena-router/scheduler/scheduler_impl.go)：先 Filter 再 Score；PD 模式两阶段（先打分 decode 取 topN，再为每个 decode 选最佳 prefill）；普通模式 topN 后随机抽选；启用 `least_request` 时调度前 `store.SyncOnFlightCounts()`。
- [factory.go](../../pkg/kthena-router/scheduler/factory.go) 注册默认插件：
  - **Score**：`gpu-usage`、`least-latency`、`least-request`、`random`、`prefix-cache`、`kvcache-aware`
  - **Filter**：`least-request`、`lora-affinity`
- 重点插件（[plugins/](../../pkg/kthena-router/scheduler/plugins)）：
  - [prefix.go](../../pkg/kthena-router/scheduler/plugins/prefix.go)：xxHash 滚动哈希分块（默认 64B/块、128 块上限）；`ModelPrefixStore` 三级 map + LRU（默认 50000 条，[cache/lru.go](../../pkg/kthena-router/scheduler/plugins/cache/lru.go)）；PostSchedule 把命中 hash 关联到选中 Pod。
  - [gpu.go](../../pkg/kthena-router/scheduler/plugins/gpu.go)：分数 = `(1 - GPUCacheUsage) * 100`。
  - `least-latency` / `least-request`：基于实时 metrics / on-flight 计数选最闲。
  - `lora-affinity`：过滤掉不持有所需 LoRA 的 Pod。
  - `kvcache-aware`：基于 kv-cache 命中度评分，配合 connectors。
  - [conf/conf.go](../../pkg/kthena-router/scheduler/plugins/conf/conf.go)：调度器 ConfigMap 解析。

## 8. 数据存储 — [datastore/](../../pkg/kthena-router/datastore)
- [store.go](../../pkg/kthena-router/datastore/store.go)（约 2000+ 行）：核心 `Store` 接口——ModelServer/Pod/ModelRoute/Gateway/HTTPRoute/InferencePool 增删改查、`MatchModelServer` 路由解析、PDGroup pod 视图、OnFlight 计数器、token tracker、fairness 入队；`Run()` 周期 scrape Pod 指标（`METRICS_SCRAPE_INTERVAL`）。
- [model_server.go](../../pkg/kthena-router/datastore/model_server.go) + [pdgroup_pods.go](../../pkg/kthena-router/datastore/pdgroup_pods.go)：按 Decode/Prefill 标签把 Pod 划分进 PD 组。
- [fairness_queue.go](../../pkg/kthena-router/datastore/fairness_queue.go)：`heap.Interface` 优先级队列；同 user FIFO，跨 user 按 `tokenWeight*tokens + requestNumWeight*requests` 排序；支持 semaphore 模式与 QPS ticker 模式，dequeue 时可刷新优先级并重建堆。
- [token_tracker.go](../../pkg/kthena-router/datastore/token_tracker.go)：每 (user,model) 秒级桶 + 累积和的滑窗（1 分钟～1 小时），input/output 加权（默认 1:2）。
- [on_flight_counter.go](../../pkg/kthena-router/datastore/on_flight_counter.go)：本地 atomic + 可选 Redis Hash（`HINCRBY/HMGET`）跨副本视图，自动钳零防 stale。

## 9. 后端引擎指标 — [backend/](../../pkg/kthena-router/backend)
- [backend.go](../../pkg/kthena-router/backend/backend.go)：`MetricsProvider` 注册表（`vLLM`/`SGLang`），统一封装 `GetPodMetrics`（合并 counter/gauge 与 histogram delta）。
- [vllm/metrics.go](../../pkg/kthena-router/backend/vllm/metrics.go)：`vllm:kv_cache_usage_perc` / `num_requests_waiting` / `num_requests_running` / `inter_token_latency_seconds` / `time_to_first_token_seconds`，默认端口 8000。
- [sglang/metrics.go](../../pkg/kthena-router/backend/sglang/metrics.go)：`sglang:token_usage` 等，默认端口 30000。
- 指标键经归一化为 `KVCacheUsage / RequestWaitingNum / RequestRunningNum / TPOT / TTFT`，供调度插件统一消费。

## 10. KV Connectors（PD 解耦）— [connectors/](../../pkg/kthena-router/connectors)
- 接口 [interface.go](../../pkg/kthena-router/connectors/interface.go)：`KVConnector { Name(); Proxy(c, reqBody, prefillAddr, decodeAddr, hooks) (int, error) }`，返回 output token 数。
- 工厂 [factory.go](../../pkg/kthena-router/connectors/factory.go)：
  - `http` / `lmcache` → [http.go](../../pkg/kthena-router/connectors/http.go)：串行 prefill → decode
  - `nixl` → [nixl.go](../../pkg/kthena-router/connectors/nixl.go)：prefill 返回 `kv_transfer_params`，decode 携回拉 KV
  - `mooncake` → [mooncake.go](../../pkg/kthena-router/connectors/mooncake.go)：复用 NIXL 实现
  - `sglang`（内部）→ [sglang.go](../../pkg/kthena-router/connectors/sglang.go)：prefill/decode 并发，`bootstrap_room`/`bootstrap_host` 走 ZMQ
- [transport.go](../../pkg/kthena-router/connectors/transport.go)：通用反代 + 流式 usage 解析 + 自动注入 `stream_options.include_usage`、prefill body `max_tokens=1`。

## 11. 可观测与调试
- [accesslog/](../../pkg/kthena-router/accesslog)：Gin 中间件 + 4 段计时（Request/Upstream Start/End/Response），JSON/Text 双格式，自动生成 `x-request-id`。
- [metrics/metrics.go](../../pkg/kthena-router/metrics/metrics.go)：`kthena_router_*` 指标族（含 prefill/decode 子段、tokens、scheduler 插件耗时、限流、active downstream/upstream、fairness queue）。
- [debug/handlers.go](../../pkg/kthena-router/debug/handlers.go)：localhost 暴露 `/debug/config_dump/{modelroutes|modelservers|pods|gateways|httproutes|inferencepools}` 只读快照。

## 12. 端到端调用链速查

```
HTTP 请求
  → AccessLogMiddleware
  → JWTAuthenticator
  → Router.HandlerFunc
      ├ Tokenizer + 输入 token 限流
      ├ FairnessQueue（按 user-token-fairness 出队）
      ├ doLoadbalance：ModelRoute(优先) | HTTPRoute(InferencePool)
      ├ Store 取候选 Pod / PDGroupPods
      ├ SchedulerImpl：Filter → Score →（PD：先 decode 后 prefill）
      ├ proxyModelEndpoint
      │     ├ 普通：transport 反代
      │     └ PD：KVConnector.Proxy（HTTP 串行 / NIXL·Mooncake KV 传参 / SGLang 并发）
      └ usage 解析 → TokenTracker → 输出限流 → Prometheus
```

---

[← 上一篇：04 自动扩缩容](./04-autoscaler.md) ｜ [返回首页](./README.md) ｜ 下一篇：[06 CLI 与部署制品](./06-cli-deploy.md)
