# 03 · 控制面 Controller Manager

[← 上一篇：02 CRD 与 API](./02-crd-api.md) ｜ [返回首页](./README.md) ｜ 下一篇：[04 自动扩缩容](./04-autoscaler.md)

---

控制面二进制 `kthena-controller-manager` 负责 reconcile CRD、运行 admission webhook、驱动自动扩缩容。入口 [cmd/kthena-controller-manager/main.go](../../cmd/kthena-controller-manager/main.go)，业务主体在 [pkg/controller](../../pkg/controller)。

## 1. 二进制入口与启动时序

[cmd/kthena-controller-manager/main.go](../../cmd/kthena-controller-manager/main.go)：

### 命令行参数（pflag）
- 通用：`--kubeconfig` / `--master` / `--leader-elect` / `--workers` / `--kube-api-qps` / `--kube-api-burst`
- 控制器开关：`--controllers="*"`（支持 `+modelserving`、`-autoscaler`；可选项 `modelserving` / `modelbooster` / `autoscaler`）
- Webhook：`--enable-webhook`（默认 true）、`--port=8443`、`--webhook-timeout=30`、`--tls-cert-file=/etc/tls/tls.crt`、`--tls-private-key-file=/etc/tls/tls.key`、`--cert-secret-name`、`--service-name`
- Debug：`--debug-port=0`（0 即关闭）

### 启动时序
1. 初始化 `klog`，捕获 `SIGINT/SIGTERM`，构建可取消 context。
2. **异步** `setupWebhook`（若 `--enable-webhook`）：
   - 创建 in-cluster `kubeClient`、`kthenaClient`。
   - CA bundle 三优先级：**Secret → 已有 cert/key 文件 → 自动签发**（`webhookcert.EnsureCertificate`）。
   - 自动写回 `ValidatingWebhookConfiguration kthena-controller-manager-validating-webhook` 与 `MutatingWebhookConfiguration kthena-controller-manager-mutating-webhook` 的 `caBundle`。
   - 注册 HTTP handlers：
     - `/validate-workload-ai-v1alpha1-modelserving`
     - `/validate/modelbooster`、`/mutate/modelbooster`
     - `/validate/autoscalingpolicy`、`/mutate/autoscalingpolicy`、`/validate/autoscalingpolicybinding`
     - `/healthz`
   - 等待 cert/key ready（≤30s）后 `ListenAndServeTLS` 在 `:8443`。
3. **同步** `controller.SetupController(ctx, cc)`。
4. 若 `--debug-port>0`，在 `localhost:debugPort` 起仅本地可访问的 debug HTTP server。

> Webhook goroutine 与 controller 主循环**无强依赖**、同时进行；webhook 提前写好 caBundle，使 apiserver 在 controller 处理资源前即可调用 webhook。

## 2. 控制器注册入口

[pkg/controller/controller.go](../../pkg/controller/controller.go) 中 `SetupController(ctx, cc Config)`：
- 构建客户端：`kubeClient`、`kthena clientset`、`volcano client`、`apiextClient`。
- 按 `cc.Controllers` 开关启用：
  - **ModelBooster** 控制器
  - **ModelServing** 控制器（含可选 LWS / LeaderWorkerSet 集成；未发现该 CRD 时降级跳过）
  - **Autoscaler** 控制器（即 [pkg/autoscaler](../../pkg/autoscaler)，详见 [04](./04-autoscaler.md)）
- **Leader Election**：基于 `coordination/v1` Lease，名为 `lease.kthena.controller-manager`，只有 leader 调用 `startControllers`。
- 启动只监听 localhost 的 Debug HTTP server。

## 3. 配置对象

[pkg/controller/config.go](../../pkg/controller/config.go) 定义 `Config`：
```
EnableLeaderElection
Workers
Kubeconfig / MasterURL
Controllers map[string]bool   // 控制器名 → 开关，实现二进制裁剪
KubeAPIQPS / KubeAPIBurst
DebugPort
```
这种 "控制器名 → 开关" 的 map 模式，让同一二进制可以按需只跑部分控制器。

## 4. 协同关系

```
ModelBooster controller ──渲染──► ModelRoute + ModelServer + ModelServing
ModelServing controller ──渲染──► ServingGroup ──► Pods（多角色）
Autoscaler controller   ──采集 Pod 指标──► 计算副本 ──patch──► ModelServing.Spec.Replicas
```

三个控制器协同形成"声明 → 部署 → 度量 → 扩缩"的闭环；webhook 在写入阶段做校验与默认值注入。

---

[← 上一篇：02 CRD 与 API](./02-crd-api.md) ｜ [返回首页](./README.md) ｜ 下一篇：[04 自动扩缩容](./04-autoscaler.md)
