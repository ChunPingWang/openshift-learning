# OpenShift/Kubernetes Level 7-10 進階實作指南

> 基於 CRC (CodeReady Containers) 環境的實戰練習結果

---

## Level 7：監控與日誌（Advanced）

### Exercise 7.1：Prometheus 監控

#### 存取 OpenShift 內建 Prometheus

```bash
# 檢視監控元件
oc get pods -n openshift-monitoring

# 取得 Prometheus Route
oc get route -n openshift-monitoring | grep prometheus
```

#### 常用 PromQL 查詢

```promql
# CPU 使用率（按 Pod）
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod)

# 記憶體使用量（按 Pod）
sum(container_memory_usage_bytes{container!=""}) by (pod)

# HTTP 請求率
sum(rate(http_requests_total[5m])) by (service)

# Pod 重啟次數（過去 1 小時）
increase(kube_pod_container_status_restarts_total[1h])

# 命名空間資源使用量
sum(container_memory_usage_bytes) by (namespace)
```

---

### Exercise 7.2：ServiceMonitor 配置

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
  labels:
    team: frontend
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
    scheme: http
  namespaceSelector:
    matchNames:
    - my-namespace
```

---

### Exercise 7.3：告警規則

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: app-alerts
spec:
  groups:
  - name: app.rules
    rules:
    - alert: HighPodRestarts
      expr: |
        increase(kube_pod_container_status_restarts_total[1h]) > 5
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} 在過去 1 小時重啟超過 5 次"

    - alert: HighCPUUsage
      expr: |
        sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod) > 0.8
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} CPU 使用率超過 80%"

    - alert: HighMemoryUsage
      expr: |
        sum(container_memory_usage_bytes{container!=""}) by (pod)
        /
        sum(container_spec_memory_limit_bytes{container!=""}) by (pod) > 0.9
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} 記憶體使用率超過 90%"
```

---

### Exercise 7.4：日誌收集

```bash
# 基本日誌檢視
oc logs <pod-name>
oc logs -f <pod-name>                    # 追蹤日誌
oc logs <pod-name> --since=1h            # 最近 1 小時
oc logs <pod-name> --tail=100            # 最後 100 行

# 多容器 Pod
oc logs <pod-name> -c <container-name>
oc logs <pod-name> --all-containers

# 之前的容器（重啟後）
oc logs <pod-name> --previous

# 使用 stern（需安裝）
stern <pod-prefix> -n <namespace>
```

---

### Exercise 7.5：Liveness、Readiness 與 Startup Probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
spec:
  containers:
  - name: app
    image: nginx
    ports:
    - containerPort: 80

    # Startup Probe - 等待應用啟動
    startupProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 0
      periodSeconds: 5
      failureThreshold: 30    # 最多等待 150 秒

    # Liveness Probe - 檢測應用是否存活
    livenessProbe:
      httpGet:
        path: /healthz
        port: 80
      initialDelaySeconds: 0
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3     # 3 次失敗後重啟

    # Readiness Probe - 檢測是否可接收流量
    readinessProbe:
      httpGet:
        path: /ready
        port: 80
      initialDelaySeconds: 0
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3     # 3 次失敗後從 Service 移除
```

#### Probe 類型

| 類型 | 說明 |
|------|------|
| `httpGet` | HTTP GET 請求 |
| `tcpSocket` | TCP 連線檢查 |
| `exec` | 執行命令（返回 0 為成功） |
| `grpc` | gRPC 健康檢查 |

---

## Level 8：CI/CD Pipeline（Tekton）

### Exercise 8.1：Tekton Task

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: build-and-test
spec:
  params:
  - name: git-url
    type: string
  - name: image-name
    type: string

  workspaces:
  - name: source

  results:
  - name: image-digest
    description: Image digest

  steps:
  - name: clone
    image: alpine/git
    script: |
      git clone $(params.git-url) $(workspaces.source.path)

  - name: build
    image: maven:3.8-openjdk-17
    script: |
      cd $(workspaces.source.path)
      mvn package -DskipTests

  - name: test
    image: maven:3.8-openjdk-17
    script: |
      cd $(workspaces.source.path)
      mvn test
```

---

### Exercise 8.2：Tekton Pipeline

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: ci-cd-pipeline
spec:
  params:
  - name: git-url
    type: string
  - name: git-revision
    type: string
    default: main
  - name: image-name
    type: string

  workspaces:
  - name: shared-workspace
  - name: docker-credentials

  tasks:
  # 1. Clone 原始碼
  - name: clone
    taskRef:
      name: git-clone
    params:
    - name: url
      value: $(params.git-url)
    - name: revision
      value: $(params.git-revision)
    workspaces:
    - name: output
      workspace: shared-workspace

  # 2. 單元測試（與 lint 並行）
  - name: test
    taskRef:
      name: maven
    runAfter: [clone]
    params:
    - name: GOALS
      value: ["test"]
    workspaces:
    - name: source
      workspace: shared-workspace

  - name: lint
    taskRef:
      name: lint
    runAfter: [clone]
    workspaces:
    - name: source
      workspace: shared-workspace

  # 3. 建置映像
  - name: build-image
    taskRef:
      name: buildah
    runAfter: [test, lint]
    params:
    - name: IMAGE
      value: $(params.image-name):$(tasks.clone.results.commit)
    workspaces:
    - name: source
      workspace: shared-workspace

  # 4. 部署到開發環境
  - name: deploy-dev
    taskRef:
      name: kubernetes-actions
    runAfter: [build-image]
    params:
    - name: script
      value: |
        kubectl set image deployment/myapp \
          app=$(params.image-name):$(tasks.clone.results.commit) \
          -n dev

  # 5. Finally 區塊（無論成功失敗都執行）
  finally:
  - name: notify
    taskRef:
      name: send-notification
    params:
    - name: status
      value: "$(tasks.status)"

  - name: cleanup
    taskRef:
      name: cleanup-workspace
```

---

### Exercise 8.3：Tekton Triggers

```yaml
# EventListener - 接收 Webhook
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: github-listener
spec:
  serviceAccountName: pipeline
  triggers:
  - name: github-push
    interceptors:
    - ref:
        name: github
      params:
      - name: secretRef
        value:
          secretName: github-webhook-secret
          secretKey: webhook-secret
      - name: eventTypes
        value: ["push"]
    bindings:
    - ref: github-push-binding
    template:
      ref: pipeline-template
---
# TriggerBinding - 解析 Webhook 資料
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: github-push-binding
spec:
  params:
  - name: git-url
    value: $(body.repository.clone_url)
  - name: git-revision
    value: $(body.after)
---
# TriggerTemplate - 產生 PipelineRun
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: pipeline-template
spec:
  params:
  - name: git-url
  - name: git-revision
  resourcetemplates:
  - apiVersion: tekton.dev/v1
    kind: PipelineRun
    metadata:
      generateName: ci-pipeline-
    spec:
      pipelineRef:
        name: ci-cd-pipeline
      params:
      - name: git-url
        value: $(tt.params.git-url)
      - name: git-revision
        value: $(tt.params.git-revision)
      workspaces:
      - name: shared-workspace
        volumeClaimTemplate:
          spec:
            accessModes: ["ReadWriteOnce"]
            resources:
              requests:
                storage: 1Gi
```

---

## Level 9：Operator 開發（Expert）

### Exercise 9.1：使用 Operator SDK 建立 Operator

```bash
# 初始化專案
operator-sdk init --domain=example.com --repo=github.com/myorg/nginx-operator

# 建立 API 和 Controller
operator-sdk create api --group=web --version=v1 --kind=NginxDeployment --resource --controller

# 生成 CRD manifests
make manifests

# 本地測試
make install  # 安裝 CRD
make run      # 執行 Controller
```

### Exercise 9.2：Reconcile 邏輯

```go
func (r *NginxDeploymentReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)

    // 1. 取得 CR
    nginx := &webv1.NginxDeployment{}
    if err := r.Get(ctx, req.NamespacedName, nginx); err != nil {
        if errors.IsNotFound(err) {
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, err
    }

    // 2. 處理 Finalizer
    if nginx.ObjectMeta.DeletionTimestamp.IsZero() {
        if !containsString(nginx.GetFinalizers(), finalizerName) {
            controllerutil.AddFinalizer(nginx, finalizerName)
            if err := r.Update(ctx, nginx); err != nil {
                return ctrl.Result{}, err
            }
        }
    } else {
        // 執行清理邏輯
        if containsString(nginx.GetFinalizers(), finalizerName) {
            if err := r.deleteExternalResources(nginx); err != nil {
                return ctrl.Result{}, err
            }
            controllerutil.RemoveFinalizer(nginx, finalizerName)
            if err := r.Update(ctx, nginx); err != nil {
                return ctrl.Result{}, err
            }
        }
        return ctrl.Result{}, nil
    }

    // 3. 建立或更新 Deployment
    deployment := &appsv1.Deployment{}
    err := r.Get(ctx, types.NamespacedName{Name: nginx.Name, Namespace: nginx.Namespace}, deployment)
    if err != nil && errors.IsNotFound(err) {
        dep := r.deploymentForNginx(nginx)
        if err := r.Create(ctx, dep); err != nil {
            return ctrl.Result{}, err
        }
        return ctrl.Result{Requeue: true}, nil
    }

    // 4. 更新 Status
    nginx.Status.ReadyReplicas = deployment.Status.ReadyReplicas
    if err := r.Status().Update(ctx, nginx); err != nil {
        return ctrl.Result{}, err
    }

    return ctrl.Result{}, nil
}
```

---

## Level 10：綜合情境題（Expert）

### Scenario 10.1：微服務架構部署清單

```
電子商務平台架構：
┌─────────────────────────────────────────────────────────────┐
│                      [Internet]                             │
│                           │                                 │
│                    [Ingress/Route]                          │
│                           │                                 │
│              ┌────────────┴────────────┐                   │
│              │                         │                    │
│        [Frontend]              [API Gateway]                │
│         (React)              (Spring Cloud)                 │
│              │                         │                    │
│              └──────────┬──────────────┘                   │
│                         │                                   │
│         ┌───────────────┼───────────────┐                  │
│         │               │               │                   │
│    [Product]       [Order]         [User]                  │
│    Service         Service         Service                  │
│         │               │               │                   │
│         └───────────────┼───────────────┘                  │
│                         │                                   │
│         ┌───────────────┼───────────────┐                  │
│         │               │               │                   │
│    [PostgreSQL]    [RabbitMQ]      [Redis]                 │
│         │               │               │                   │
│    [PVC]           [PVC]          [PVC]                    │
└─────────────────────────────────────────────────────────────┘
```

### Scenario 10.3：藍綠部署腳本

```bash
#!/bin/bash
# Blue-Green Deployment Script

NAMESPACE="production"
APP_NAME="myapp"
NEW_VERSION="v2"

# 1. 部署 Green 環境
oc apply -f deployment-green.yaml

# 2. 等待 Green 就緒
oc rollout status deployment/${APP_NAME}-green -n $NAMESPACE

# 3. 驗證 Green 健康
HEALTH_CHECK=$(curl -s -o /dev/null -w "%{http_code}" http://${APP_NAME}-green.${NAMESPACE}.svc:8080/health)
if [ "$HEALTH_CHECK" != "200" ]; then
    echo "Green deployment health check failed"
    exit 1
fi

# 4. 切換流量到 Green
oc patch service ${APP_NAME} -n $NAMESPACE \
  -p '{"spec":{"selector":{"version":"green"}}}'

# 5. 驗證切換成功
sleep 10
PROD_CHECK=$(curl -s http://${APP_NAME}.${NAMESPACE}.svc:8080/version)
if [ "$PROD_CHECK" != "$NEW_VERSION" ]; then
    echo "Traffic switch failed, rolling back..."
    oc patch service ${APP_NAME} -n $NAMESPACE \
      -p '{"spec":{"selector":{"version":"blue"}}}'
    exit 1
fi

# 6. 清理舊的 Blue 環境（可選）
# oc delete deployment ${APP_NAME}-blue -n $NAMESPACE

echo "Blue-Green deployment completed successfully"
```

---

## 常用指令速查表

### 監控相關

```bash
# 檢視 Pod 資源使用
oc adm top pods
oc adm top nodes

# 檢視事件
oc get events --sort-by='.lastTimestamp'

# 除錯節點
oc debug node/<node-name>
```

### Tekton 相關

```bash
# 列出 Pipeline 資源
oc get pipelines
oc get pipelineruns
oc get tasks
oc get taskruns

# 檢視 PipelineRun 日誌
tkn pipelinerun logs <name>

# 重新執行 Pipeline
tkn pipeline start <pipeline-name>
```

### Operator 相關

```bash
# 檢視 Operator 狀態
oc get csv -A
oc get subscriptions -A

# 檢視 CRD
oc get crd
oc explain <crd-name>.spec
```

---

*文件產生時間：2026-02-04*
*測試環境：CRC 2.57.0 / OpenShift 4.x*

**注意：** Level 7-10 的部分功能（如 Prometheus、Tekton）可能需要額外安裝 Operator 才能使用。
