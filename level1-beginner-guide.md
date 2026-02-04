# OpenShift/Kubernetes Level 1 初學者實作指南

> 基於 CRC (CodeReady Containers) 環境的實戰練習結果

---

## 環境資訊

| 項目 | 值 |
|------|-----|
| Web Console | https://console-openshift-console.apps-crc.testing |
| API Server | https://api.crc.testing:6443 |
| 管理員帳號 | kubeadmin |
| 管理員密碼 | *依安裝結果而定* |
| 一般使用者 | developer / developer |

> **⚠️ 重要提醒：** `kubeadmin` 的密碼在每次 CRC 安裝時都會自動產生，每個環境的密碼都不同。請使用以下指令取得您環境的實際密碼：
> ```bash
> crc console --credentials
> ```

---

## 環境準備

### 設定 oc 命令列工具

```bash
# 設定環境變數
eval $(crc oc-env)

# 取得登入憑證
crc console --credentials

# 以管理員身份登入（請替換 <password> 為您的實際密碼）
oc login -u kubeadmin -p <password> https://api.crc.testing:6443
```

**預期輸出：**
```
Login successful.
You have access to 65 projects, the list has been suppressed. You can list all projects with 'oc projects'
Using project "default".
```

### 建立練習用專案

```bash
oc new-project exercises
```

---

## Exercise 1.1：專案與命名空間管理

### 題目
1. 建立名為 `dev-team-a` 的專案，描述為 "Development Team A Project"
2. 建立名為 `dev-team-b` 的專案
3. 列出所有專案
4. 切換到 `dev-team-a` 專案
5. 為 `dev-team-a` 專案新增標籤 `environment=development`
6. 刪除 `dev-team-b` 專案

### 指令與結果

#### 1. 建立專案 dev-team-a（含描述）

```bash
oc new-project dev-team-a --description="Development Team A Project" --display-name="Dev Team A"
```

**輸出：**
```
Now using project "dev-team-a" on server "https://api.crc.testing:6443".

You can add applications to this project with the 'new-app' command. For example, try:
    oc new-app rails-postgresql-example
```

#### 2. 建立專案 dev-team-b

```bash
oc new-project dev-team-b
```

**輸出：**
```
Now using project "dev-team-b" on server "https://api.crc.testing:6443".
```

#### 3. 列出所有專案

```bash
oc get projects
```

**輸出（過濾後）：**
```
NAME         DISPLAY NAME   STATUS
dev-team-a   Dev Team A     Active
dev-team-b                  Active
```

#### 4. 切換專案

```bash
oc project dev-team-a
```

**輸出：**
```
Now using project "dev-team-a" on server "https://api.crc.testing:6443".
```

#### 5. 新增標籤

```bash
oc label namespace dev-team-a environment=development
```

**輸出：**
```
namespace/dev-team-a labeled
```

#### 6. 刪除專案

```bash
oc delete project dev-team-b
```

**輸出：**
```
project.project.openshift.io "dev-team-b" deleted
```

### 驗證

```bash
oc get project dev-team-a --show-labels
```

**輸出：**
```
NAME         DISPLAY NAME   STATUS   LABELS
dev-team-a   Dev Team A     Active   environment=development,kubernetes.io/metadata.name=dev-team-a,...
```

---

## Exercise 1.2：Pod 基本操作

### 題目
1. 使用 `nginx:1.25` 映像建立一個名為 `web-pod` 的 Pod
2. 檢視 Pod 的詳細資訊
3. 檢視 Pod 的日誌
4. 進入 Pod 執行 `nginx -v` 指令
5. 將本機 8080 端口轉發到 Pod 的 80 端口
6. 刪除該 Pod

### 指令與結果

#### 1. 建立 Pod

```bash
oc run web-pod --image=nginx:1.25
```

**輸出：**
```
pod/web-pod created
```

等待 Pod 就緒：
```bash
oc wait --for=condition=Ready pod/web-pod --timeout=120s
```

**輸出：**
```
pod/web-pod condition met
```

#### 2. 檢視 Pod 詳細資訊

```bash
oc get pod web-pod -o wide
```

**輸出：**
```
NAME      READY   STATUS    RESTARTS   AGE   IP             NODE   NOMINATED NODE   READINESS GATES
web-pod   1/1     Running   0          17s   10.217.0.115   crc    <none>           <none>
```

更詳細的資訊：
```bash
oc describe pod web-pod
```

#### 3. 檢視 Pod 日誌

```bash
oc logs web-pod
```

**輸出：**
```
2026/02/04 11:43:37 [notice] 1#1: nginx/1.25.5
2026/02/04 11:43:37 [notice] 1#1: built by gcc 12.2.0 (Debian 12.2.0-14)
2026/02/04 11:43:37 [notice] 1#1: OS: Linux 5.14.0-570.66.1.el9_6.x86_64
2026/02/04 11:43:37 [notice] 1#1: start worker processes
```

常用日誌選項：
```bash
oc logs web-pod --tail=10        # 只顯示最後 10 行
oc logs web-pod -f               # 持續追蹤日誌
oc logs web-pod --since=1h       # 顯示最近 1 小時的日誌
```

#### 4. 在 Pod 內執行指令

```bash
oc exec web-pod -- nginx -v
```

**輸出：**
```
nginx version: nginx/1.25.5
```

進入互動式 shell：
```bash
oc exec -it web-pod -- /bin/bash
```

#### 5. 端口轉發

```bash
oc port-forward web-pod 8080:80
```

**說明：** 此指令會將本機的 8080 端口轉發到 Pod 的 80 端口，可透過 http://localhost:8080 存取 nginx。

#### 6. 刪除 Pod

```bash
oc delete pod web-pod
```

**輸出：**
```
pod "web-pod" deleted
```

---

## Exercise 1.3：使用 YAML 建立資源

### 題目
建立一個 YAML 檔案，定義以下 Pod：
- 名稱：`multi-container-pod`
- 包含兩個容器：
  - 容器 1：名稱 `nginx`，映像 `nginx:1.25`，端口 80
  - 容器 2：名稱 `sidecar`，映像 `busybox`，執行 `sleep 3600`
- 標籤：`app=web`, `tier=frontend`

### 指令與結果

#### 1. 建立 YAML 檔案

```yaml
# multi-container-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
  labels:
    app: web
    tier: frontend
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
  - name: sidecar
    image: busybox
    command: ["sleep", "3600"]
```

#### 2. 套用 YAML

```bash
oc apply -f multi-container-pod.yaml
```

**輸出：**
```
pod/multi-container-pod created
```

#### 3. 等待 Pod 就緒

```bash
oc wait --for=condition=Ready pod/multi-container-pod --timeout=120s
```

**輸出：**
```
pod/multi-container-pod condition met
```

### 驗證

```bash
oc get pod multi-container-pod --show-labels
```

**輸出：**
```
NAME                  READY   STATUS    RESTARTS   AGE   LABELS
multi-container-pod   2/2     Running   0          7s    app=web,tier=frontend
```

檢視容器列表：
```bash
oc get pod multi-container-pod -o jsonpath='{.spec.containers[*].name}'
```

**輸出：**
```
nginx sidecar
```

---

## Exercise 1.4：標籤與選擇器

### 題目
1. 建立 3 個 Pod，使用不同的標籤
2. 使用標籤選擇器列出特定 Pod
3. 新增和移除標籤

### 指令與結果

#### 1. 建立帶標籤的 Pod

```bash
oc run pod-1 --image=busybox --labels="app=frontend,env=dev" -- sleep 3600
oc run pod-2 --image=busybox --labels="app=backend,env=dev" -- sleep 3600
oc run pod-3 --image=busybox --labels="app=frontend,env=prod" -- sleep 3600
```

檢視所有 Pod：
```bash
oc get pods --show-labels
```

**輸出：**
```
NAME    READY   STATUS    RESTARTS   AGE   LABELS
pod-1   1/1     Running   0          5s    app=frontend,env=dev
pod-2   1/1     Running   0          5s    app=backend,env=dev
pod-3   1/1     Running   0          5s    app=frontend,env=prod
```

#### 2. 使用標籤選擇器

列出 `app=frontend` 的 Pod：
```bash
oc get pods -l app=frontend
```

**輸出：**
```
NAME    READY   STATUS    RESTARTS   AGE
pod-1   1/1     Running   0          5s
pod-3   1/1     Running   0          5s
```

列出 `env=dev` 的 Pod：
```bash
oc get pods -l env=dev
```

**輸出：**
```
NAME    READY   STATUS    RESTARTS   AGE
pod-1   1/1     Running   0          5s
pod-2   1/1     Running   0          5s
```

列出同時滿足 `app=frontend` 和 `env=dev` 的 Pod：
```bash
oc get pods -l app=frontend,env=dev
```

**輸出：**
```
NAME    READY   STATUS    RESTARTS   AGE
pod-1   1/1     Running   0          5s
```

#### 3. 新增標籤

```bash
oc label pod pod-1 version=v1
```

**輸出：**
```
pod/pod-1 labeled
```

驗證：
```bash
oc get pod pod-1 --show-labels
```

**輸出：**
```
NAME    READY   STATUS    RESTARTS   AGE   LABELS
pod-1   1/1     Running   0          5s    app=frontend,env=dev,version=v1
```

#### 4. 移除標籤

```bash
oc label pod pod-2 env-
```

**輸出：**
```
pod/pod-2 unlabeled
```

驗證：
```bash
oc get pod pod-2 --show-labels
```

**輸出：**
```
NAME    READY   STATUS    RESTARTS   AGE   LABELS
pod-2   1/1     Running   0          6s    app=backend
```

---

## Exercise 1.5：資源查詢與輸出格式

### 題目
1. 以 YAML 格式輸出 Pod 定義
2. 以 JSON 格式輸出所有 Pod
3. 使用 JSONPath 取得 Pod 名稱和 IP
4. 使用自訂欄位輸出
5. 匯出 Deployment 為 YAML

### 指令與結果

#### 1. YAML 格式輸出

```bash
oc get pod pod-1 -o yaml
```

**輸出（節錄）：**
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: frontend
    env: dev
    version: v1
  name: pod-1
  namespace: exercises
spec:
  containers:
  - command:
    - sleep
    - "3600"
    image: busybox
    name: pod-1
```

#### 2. JSON 格式輸出

```bash
oc get pods -o json
```

取得 Pod 數量：
```bash
oc get pods -o json | jq '.items | length'
```

**輸出：**
```
6
```

#### 3. JSONPath 查詢

取得所有 Pod 名稱：
```bash
oc get pods -o jsonpath='{.items[*].metadata.name}'
```

**輸出：**
```
multi-container-pod nginx-demo-xxx pod-1 pod-2 pod-3
```

取得 Pod 名稱和 IP：
```bash
oc get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'
```

**輸出：**
```
multi-container-pod     10.217.0.116
pod-1                   10.217.0.117
pod-2                   10.217.0.118
pod-3                   10.217.0.119
```

#### 4. 自訂欄位輸出

```bash
oc get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,IP:.status.podIP
```

**輸出：**
```
NAME                  STATUS    IP
multi-container-pod   Running   10.217.0.116
pod-1                 Running   10.217.0.117
pod-2                 Running   10.217.0.118
pod-3                 Running   10.217.0.119
```

#### 5. 匯出 Deployment 為 YAML

先建立 Deployment：
```bash
oc create deployment nginx-demo --image=nginx:1.25 --replicas=2
```

匯出為 YAML：
```bash
oc get deployment nginx-demo -o yaml > nginx-demo-export.yaml
```

**檔案內容（節錄）：**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-demo
  name: nginx-demo
  namespace: exercises
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
      - image: nginx:1.25
        name: nginx
```

#### 6. Wide 輸出格式

```bash
oc get pods -o wide
```

**輸出：**
```
NAME                          READY   STATUS    RESTARTS   AGE   IP             NODE   NOMINATED NODE   READINESS GATES
multi-container-pod           2/2     Running   0          35s   10.217.0.116   crc    <none>           <none>
pod-1                         1/1     Running   0          20s   10.217.0.117   crc    <none>           <none>
pod-2                         1/1     Running   0          20s   10.217.0.118   crc    <none>           <none>
pod-3                         1/1     Running   0          20s   10.217.0.119   crc    <none>           <none>
```

---

## 常用指令速查表

### 專案管理

| 指令 | 說明 |
|------|------|
| `oc new-project <name>` | 建立新專案 |
| `oc project <name>` | 切換專案 |
| `oc get projects` | 列出所有專案 |
| `oc delete project <name>` | 刪除專案 |
| `oc label namespace <name> key=value` | 為專案加標籤 |

### Pod 操作

| 指令 | 說明 |
|------|------|
| `oc run <name> --image=<image>` | 建立 Pod |
| `oc get pods` | 列出 Pod |
| `oc get pods -o wide` | 列出 Pod（含更多資訊） |
| `oc describe pod <name>` | 顯示 Pod 詳情 |
| `oc logs <name>` | 檢視日誌 |
| `oc logs -f <name>` | 追蹤日誌 |
| `oc exec <name> -- <command>` | 執行指令 |
| `oc exec -it <name> -- /bin/bash` | 進入容器 |
| `oc delete pod <name>` | 刪除 Pod |

### 標籤操作

| 指令 | 說明 |
|------|------|
| `oc get pods --show-labels` | 顯示標籤 |
| `oc get pods -l key=value` | 依標籤篩選 |
| `oc label pod <name> key=value` | 新增標籤 |
| `oc label pod <name> key-` | 移除標籤 |

### 輸出格式

| 指令 | 說明 |
|------|------|
| `-o yaml` | YAML 格式 |
| `-o json` | JSON 格式 |
| `-o wide` | 擴展資訊 |
| `-o jsonpath='{...}'` | JSONPath 查詢 |
| `-o custom-columns=...` | 自訂欄位 |

---

## 注意事項

### Pod Security 警告

在 OpenShift 4.x 中，你可能會看到類似以下的警告：
```
Warning: would violate PodSecurity "restricted:latest": allowPrivilegeEscalation != false...
```

這是因為 OpenShift 預設啟用了 Pod Security Standards。在生產環境中，建議為 Pod 設定適當的安全上下文：

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
          - ALL
```

### 清理資源

練習完成後，記得清理資源：
```bash
oc delete pod --all
oc delete deployment --all
oc delete project dev-team-a
```

---

## 下一步

完成 Level 1 後，建議繼續練習：
- **Level 2**：Deployment、ReplicaSet、StatefulSet
- **Level 3**：ConfigMap、Secret
- **Level 4**：Service、Route、NetworkPolicy

---

*文件產生時間：2026-02-04*
*測試環境：CRC 2.57.0 / OpenShift 4.x*
