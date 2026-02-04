# OpenShift/Kubernetes Level 3 配置管理實作指南

> 基於 OpenShift Local（原 CodeReady Containers）環境的實戰練習結果

> **⚠️ 重要提醒：** `kubeadmin` 的密碼在每次 OpenShift Local 安裝時都會自動產生，每個環境的密碼都不同。請使用 `crc console --credentials` 指令取得您環境的實際密碼。

---

## Exercise 3.1：ConfigMap 操作

### 題目
1. 從字面值建立 ConfigMap
2. 從檔案建立 ConfigMap
3. 將 ConfigMap 掛載為環境變數
4. 將 ConfigMap 掛載為 Volume

### 指令與結果

#### 1. 從字面值建立 ConfigMap

```bash
oc create configmap app-config \
  --from-literal=database_host=postgres.default.svc \
  --from-literal=database_port=5432 \
  --from-literal=log_level=INFO
```

**輸出：**
```yaml
apiVersion: v1
data:
  database_host: postgres.default.svc
  database_port: "5432"
  log_level: INFO
kind: ConfigMap
metadata:
  name: app-config
  namespace: exercises
```

#### 2. 從檔案建立 ConfigMap

建立設定檔：
```bash
cat << 'EOF' > application.properties
# Application Configuration
app.name=MyApp
app.version=1.0.0
server.port=8080
database.url=jdbc:postgresql://postgres:5432/mydb
database.pool.size=10
logging.level=INFO
EOF
```

建立 ConfigMap：
```bash
oc create configmap app-properties --from-file=application.properties
```

#### 3. ConfigMap 作為環境變數

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-env-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "env | grep -E 'database|log' && sleep 3600"]
    envFrom:
    - configMapRef:
        name: app-config
```

**Pod 內的環境變數：**
```
database_port=5432
log_level=INFO
database_host=postgres.default.svc
```

#### 4. ConfigMap 作為 Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-volume-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "cat /etc/config/application.properties && sleep 3600"]
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-properties
```

---

## Exercise 3.2：Secret 管理

### 題目
1. 建立 Opaque Secret（資料庫認證）
2. 建立 TLS Secret
3. 建立 Docker Registry Secret
4. 將 Secret 掛載為特定環境變數
5. 將 Secret 掛載為檔案
6. 使用 stringData 建立 Secret

### 指令與結果

#### 1. 建立 Opaque Secret

```bash
oc create secret generic db-secret \
  --from-literal=username=dbadmin \
  --from-literal=password=SuperSecretPass123
```

**驗證（解碼 base64）：**
```bash
oc get secret db-secret -o jsonpath='{.data.password}' | base64 -d
```

**輸出：**
```
SuperSecretPass123
```

#### 2. 建立 TLS Secret

```bash
# 產生自簽憑證
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=myapp.example.com"

# 建立 TLS Secret
oc create secret tls tls-secret \
  --cert=tls.crt \
  --key=tls.key
```

#### 3. 建立 Docker Registry Secret

```bash
oc create secret docker-registry registry-secret \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=user@example.com
```

#### 4. Secret 作為特定環境變數

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo 'DB_USER='$DB_USER && sleep 3600"]
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```

#### 5. Secret 作為檔案

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "ls -la /etc/secrets && cat /etc/secrets/username && sleep 3600"]
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-secret
```

**輸出（掛載為檔案）：**
```
total 0
lrwxrwxrwx  1 root root  15 Feb  4 12:04 password -> ..data/password
lrwxrwxrwx  1 root root  15 Feb  4 12:04 username -> ..data/username
```

#### 6. 使用 stringData

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: string-secret
type: Opaque
stringData:
  api-key: my-api-key-12345
  config.yaml: |
    host: api.example.com
    port: 443
    ssl: true
```

**重點：** `stringData` 允許直接輸入明文，Kubernetes 會自動進行 base64 編碼存儲。

---

## Exercise 3.3：環境變數進階

### 題目
建立 Pod 包含多種環境變數來源：
1. 靜態值
2. ConfigMap
3. Secret
4. Downward API（Pod 欄位）
5. 容器資源欄位

### YAML 定義

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-demo
  labels:
    app: demo
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "env | sort && sleep 3600"]
    resources:
      limits:
        cpu: 200m
        memory: 128Mi
      requests:
        cpu: 100m
        memory: 64Mi
    env:
    # 1. 靜態值
    - name: STATIC_VALUE
      value: "hello-world"

    # 2. 從 ConfigMap
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: log_level

    # 3. 從 Secret
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password

    # 4. Downward API - Pod 欄位
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName

    # 5. 容器資源欄位
    - name: CPU_LIMIT
      valueFrom:
        resourceFieldRef:
          containerName: app
          resource: limits.cpu
    - name: MEMORY_LIMIT
      valueFrom:
        resourceFieldRef:
          containerName: app
          resource: limits.memory
```

### 執行結果

```
CPU_LIMIT=1
DB_PASSWORD=SuperSecretPass123
LOG_LEVEL=INFO
MEMORY_LIMIT=134217728
NODE_NAME=crc
POD_IP=10.217.0.163
POD_NAME=env-demo
POD_NAMESPACE=exercises
STATIC_VALUE=hello-world
```

---

## Exercise 3.4：ConfigMap 熱更新

### 題目
1. 建立 ConfigMap 並掛載到 Pod
2. 更新 ConfigMap 內容
3. 驗證 Pod 內檔案自動更新

### 指令與結果

#### 1. 建立 ConfigMap

```bash
oc create configmap hot-config \
  --from-literal=message="Original Message" \
  --from-literal=version=v1
```

#### 2. 建立 Pod（Volume 掛載）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hot-update-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "while true; do cat /etc/config/message; sleep 10; done"]
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: hot-config
```

#### 3. 更新 ConfigMap

```bash
oc patch configmap hot-config -p '{"data":{"message":"Updated Message!","version":"v2"}}'
```

#### 4. 驗證自動更新

等待 30-60 秒後，Pod 內的檔案會自動更新。

### 重要說明

| 掛載方式 | 自動更新 | 說明 |
|---------|---------|------|
| Volume 掛載 | ✓ 是 | 有 30-60 秒延遲 |
| 環境變數 | ✗ 否 | 需要重啟 Pod |

---

## 常用指令速查表

### ConfigMap 操作

| 指令 | 說明 |
|------|------|
| `oc create configmap <name> --from-literal=k=v` | 從字面值建立 |
| `oc create configmap <name> --from-file=<file>` | 從檔案建立 |
| `oc get configmap <name> -o yaml` | 檢視內容 |
| `oc edit configmap <name>` | 編輯 ConfigMap |
| `oc delete configmap <name>` | 刪除 |

### Secret 操作

| 指令 | 說明 |
|------|------|
| `oc create secret generic <name> --from-literal=k=v` | 建立 Opaque Secret |
| `oc create secret tls <name> --cert=<file> --key=<file>` | 建立 TLS Secret |
| `oc create secret docker-registry <name> --docker-server=...` | 建立 Registry Secret |
| `oc get secret <name> -o jsonpath='{.data.key}' \| base64 -d` | 解碼 Secret |

### 驗證指令

```bash
# 檢視 Pod 內的環境變數
oc exec <pod> -- env

# 檢視 Pod 內的掛載檔案
oc exec <pod> -- cat /etc/config/filename

# 檢視 Pod 的 Volume 掛載
oc describe pod <pod> | grep -A10 "Mounts:"
```

---

## Secret 類型參考

| 類型 | 說明 |
|------|------|
| `Opaque` | 通用 Secret（預設） |
| `kubernetes.io/tls` | TLS 憑證 |
| `kubernetes.io/dockerconfigjson` | Docker Registry 認證 |
| `kubernetes.io/basic-auth` | 基本認證 |
| `kubernetes.io/ssh-auth` | SSH 金鑰 |
| `kubernetes.io/service-account-token` | ServiceAccount Token |

---

*文件產生時間：2026-02-04*
*測試環境：OpenShift Local 2.57.0 / OpenShift 4.x*
