# OpenShift/Kubernetes Level 6 安全性實作指南

> 基於 OpenShift Local（原 CodeReady Containers）環境的實戰練習結果

> **⚠️ 重要提醒：** `kubeadmin` 的密碼在每次 OpenShift Local 安裝時都會自動產生，每個環境的密碼都不同。請使用 `crc console --credentials` 指令取得您環境的實際密碼。

---

## Exercise 6.1：RBAC 配置

### 題目
1. 建立 ServiceAccount
2. 建立 Role 授予讀取權限
3. 建立 RoleBinding 綁定
4. 建立 ClusterRole 和 ClusterRoleBinding
5. 測試權限

### 指令與結果

#### 1. 建立 ServiceAccount

```bash
oc create serviceaccount app-sa
```

#### 2. 建立 Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "watch", "list"]
```

#### 3. 建立 RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
subjects:
- kind: ServiceAccount
  name: app-sa
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

#### 4. 測試權限

```bash
# 測試 ServiceAccount 權限
oc auth can-i get pods --as=system:serviceaccount:exercises:app-sa
# yes

oc auth can-i create pods --as=system:serviceaccount:exercises:app-sa
# no

oc auth can-i get deployments --as=system:serviceaccount:exercises:app-sa
# yes

oc auth can-i delete deployments --as=system:serviceaccount:exercises:app-sa
# no
```

---

## Exercise 6.2：Security Context Constraints（OpenShift）

### 題目
1. 列出所有 SCC
2. 檢視 restricted SCC
3. 測試需要 root 權限的 Pod
4. 授予 anyuid SCC

### 常見 SCC 說明

| SCC | 說明 | 使用場景 |
|-----|------|----------|
| `restricted-v2` | 最嚴格的 SCC（預設） | 一般應用 |
| `anyuid` | 允許任意 UID | 需要特定用戶的應用 |
| `nonroot` | 必須非 root | 安全性要求的應用 |
| `privileged` | 完全特權 | 系統級元件 |
| `hostnetwork` | 允許主機網路 | 網路監控工具 |

### 指令與結果

```bash
# 列出所有 SCC
oc get scc

# 檢視特定 SCC
oc describe scc restricted-v2

# 授予 SCC 給 ServiceAccount
oc adm policy add-scc-to-user anyuid -z app-sa

# 移除 SCC
oc adm policy remove-scc-from-user anyuid -z app-sa

# 檢查 Pod 使用的 SCC
oc get pod <pod-name> -o jsonpath='{.metadata.annotations.openshift\.io/scc}'
```

### 使用 anyuid SCC 的 Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: anyuid-pod
spec:
  serviceAccountName: app-sa  # 已授予 anyuid SCC
  containers:
  - name: app
    image: nginx
```

---

## Exercise 6.3：Pod Security Standards

### 題目
建立符合安全最佳實踐的 Pod

### 安全 Pod 範例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  # Pod 層級安全設定
  securityContext:
    runAsNonRoot: true          # 必須以非 root 執行
    runAsUser: 1000             # 指定 UID
    runAsGroup: 1000            # 指定 GID
    fsGroup: 1000               # Volume 掛載的 GID
    seccompProfile:
      type: RuntimeDefault      # 系統呼叫過濾

  containers:
  - name: app
    image: nginx
    # 容器層級安全設定
    securityContext:
      allowPrivilegeEscalation: false   # 禁止提權
      readOnlyRootFilesystem: true      # 唯讀根檔案系統
      capabilities:
        drop:
        - ALL                           # 移除所有 Linux capabilities

    # 因為唯讀根檔案系統，需要掛載可寫目錄
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /var/cache/nginx
    - name: run
      mountPath: /var/run

  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}
```

### Pod Security Admission 標籤

```yaml
# Namespace 層級的 Pod 安全標籤
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
```

| 模式 | 說明 |
|------|------|
| `enforce` | 違反時拒絕 Pod 建立 |
| `warn` | 違反時顯示警告但允許 |
| `audit` | 違反時記錄審計日誌 |

| 等級 | 說明 |
|------|------|
| `privileged` | 完全不限制 |
| `baseline` | 基本限制 |
| `restricted` | 嚴格限制（最安全） |

---

## Exercise 6.5：網路安全（零信任模型）

### 題目
實作零信任網路架構：
1. 預設拒絕所有流量
2. 只允許必要的服務間通訊

### 零信任網路政策

#### 1. 預設拒絕所有流量

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

#### 2. 允許 DNS 查詢

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: openshift-dns
    ports:
    - protocol: UDP
      port: 53
```

#### 3. 分層網路架構

```
┌─────────────────────────────────────────────────────────────┐
│                    Zero Trust Network                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [Internet] → [Ingress Controller]                          │
│                       │                                     │
│                       ▼                                     │
│              ┌─────────────────┐                           │
│              │    Frontend     │ (tier=frontend)           │
│              │   (port 80)     │                           │
│              └────────┬────────┘                           │
│                       │ port 8080                          │
│                       ▼                                     │
│              ┌─────────────────┐                           │
│              │    Backend      │ (tier=backend)            │
│              │  (port 8080)    │                           │
│              └────────┬────────┘                           │
│                       │ port 5432                          │
│                       ▼                                     │
│              ┌─────────────────┐                           │
│              │    Database     │ (tier=database)           │
│              │  (port 5432)    │                           │
│              └─────────────────┘                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 常用指令速查表

### RBAC 操作

| 指令 | 說明 |
|------|------|
| `oc create sa <name>` | 建立 ServiceAccount |
| `oc get roles` | 列出 Roles |
| `oc get rolebindings` | 列出 RoleBindings |
| `oc auth can-i <verb> <resource>` | 檢查權限 |
| `oc auth can-i --list` | 列出所有權限 |

### SCC 操作

| 指令 | 說明 |
|------|------|
| `oc get scc` | 列出所有 SCC |
| `oc describe scc <name>` | 檢視 SCC 詳情 |
| `oc adm policy add-scc-to-user <scc> -z <sa>` | 授予 SCC |
| `oc adm policy remove-scc-from-user <scc> -z <sa>` | 移除 SCC |
| `oc adm policy who-can <verb> <resource>` | 查看誰有權限 |

### 安全檢查

```bash
# 檢查 Pod 的安全上下文
oc get pod <pod> -o jsonpath='{.spec.securityContext}'

# 檢查容器的安全上下文
oc get pod <pod> -o jsonpath='{.spec.containers[0].securityContext}'

# 檢查 Pod 使用的 SCC
oc get pod <pod> -o jsonpath='{.metadata.annotations.openshift\.io/scc}'

# 模擬特定身份執行
oc auth can-i --list --as=system:serviceaccount:namespace:sa-name
```

---

## Security Context 設定對照表

| 設定 | 說明 | 建議值 |
|------|------|--------|
| `runAsNonRoot` | 強制非 root 執行 | `true` |
| `runAsUser` | 指定用戶 ID | 非 0 的 UID |
| `allowPrivilegeEscalation` | 允許提權 | `false` |
| `readOnlyRootFilesystem` | 唯讀根檔案系統 | `true` |
| `capabilities.drop` | 移除 capabilities | `["ALL"]` |
| `seccompProfile.type` | Seccomp 設定檔 | `RuntimeDefault` |

---

*文件產生時間：2026-02-04*
*測試環境：OpenShift Local 2.57.0 / OpenShift 4.x*
