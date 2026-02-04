# OpenShift / Kubernetes 實戰練習題集

> 使用 CRC 環境進行實作練習

> **⚠️ 重要提醒：** `kubeadmin` 的密碼在每次 CRC 安裝時都會自動產生，每個環境的密碼都不同。請使用 `crc console --credentials` 指令取得您環境的實際密碼。

---

## 目錄

1. [練習環境準備](#練習環境準備)
2. [Level 1：基礎操作（Beginner）](#level-1基礎操作beginner)
3. [Level 2：應用部署（Intermediate）](#level-2應用部署intermediate)
4. [Level 3：配置管理（Intermediate）](#level-3配置管理intermediate)
5. [Level 4：網路與服務（Intermediate）](#level-4網路與服務intermediate)
6. [Level 5：儲存管理（Intermediate）](#level-5儲存管理intermediate)
7. [Level 6：安全性（Advanced）](#level-6安全性advanced)
8. [Level 7：監控與日誌（Advanced）](#level-7監控與日誌advanced)
9. [Level 8：CI/CD Pipeline（Advanced）](#level-8cicd-pipelineadvanced)
10. [Level 9：Operator 開發（Expert）](#level-9operator-開發expert)
11. [Level 10：綜合情境題（Expert）](#level-10綜合情境題expert)
12. [模擬考題：EX280 風格](#模擬考題ex280-風格)
13. [解答與提示](#解答與提示)

---

## 練習環境準備

### 啟動 CRC 環境

```bash
# 啟動 CRC
crc start

# 設定環境變數
eval $(crc oc-env)

# 以管理員登入
oc login -u kubeadmin -p $(crc console --credentials | grep kubeadmin | awk -F"'" '{print $2}') https://api.crc.testing:6443

# 建立練習用專案
oc new-project exercises
```

### 練習資源庫

```bash
# Clone 練習資源（選用）
git clone https://github.com/openshift/origin.git ~/openshift-examples
```

---

## Level 1：基礎操作（Beginner）

### Exercise 1.1：專案與命名空間管理

**題目：**
1. 建立名為 `dev-team-a` 的專案，描述為 "Development Team A Project"
2. 建立名為 `dev-team-b` 的專案
3. 列出所有專案
4. 切換到 `dev-team-a` 專案
5. 為 `dev-team-a` 專案新增標籤 `environment=development`
6. 刪除 `dev-team-b` 專案

**驗證：**
```bash
# 確認專案存在且有正確標籤
oc get project dev-team-a --show-labels
```

---

### Exercise 1.2：Pod 基本操作

**題目：**
1. 在 `exercises` 專案中，使用 `nginx:1.25` 映像建立一個名為 `web-pod` 的 Pod
2. 檢視 Pod 的詳細資訊
3. 檢視 Pod 的日誌
4. 進入 Pod 執行 `nginx -v` 指令
5. 將本機 8080 端口轉發到 Pod 的 80 端口
6. 刪除該 Pod

**驗證：**
```bash
# Pod 應該處於 Running 狀態
oc get pod web-pod -o wide
```

---

### Exercise 1.3：使用 YAML 建立資源

**題目：**
建立一個 YAML 檔案，定義以下 Pod：
- 名稱：`multi-container-pod`
- 包含兩個容器：
  - 容器 1：名稱 `nginx`，映像 `nginx:1.25`，端口 80
  - 容器 2：名稱 `sidecar`，映像 `busybox`，執行 `sleep 3600`
- 為 Pod 新增標籤：`app=web`, `tier=frontend`

**提示：**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
  labels:
    # 填入標籤
spec:
  containers:
  # 填入容器定義
```

---

### Exercise 1.4：標籤與選擇器

**題目：**
1. 建立 3 個 Pod，使用不同的標籤：
   - `pod-1`: `app=frontend`, `env=dev`
   - `pod-2`: `app=backend`, `env=dev`
   - `pod-3`: `app=frontend`, `env=prod`
2. 使用標籤選擇器列出所有 `app=frontend` 的 Pod
3. 使用標籤選擇器列出所有 `env=dev` 的 Pod
4. 使用標籤選擇器列出 `app=frontend` 且 `env=dev` 的 Pod
5. 為 `pod-1` 新增標籤 `version=v1`
6. 移除 `pod-2` 的 `env` 標籤

**驗證：**
```bash
oc get pods -l app=frontend
oc get pods -l env=dev
oc get pods -l app=frontend,env=dev
```

---

### Exercise 1.5：資源查詢與輸出格式

**題目：**
1. 以 YAML 格式輸出 `web-pod` 的定義
2. 以 JSON 格式輸出所有 Pod
3. 使用 JSONPath 取得所有 Pod 的名稱
4. 使用 JSONPath 取得所有 Pod 的 IP 地址
5. 使用自訂欄位輸出 Pod 名稱和狀態
6. 將某個 Deployment 的定義匯出為 YAML 檔案

**提示：**
```bash
oc get pods -o jsonpath='{.items[*].metadata.name}'
oc get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase
```

---

## Level 2：應用部署（Intermediate）

### Exercise 2.1：Deployment 管理

**題目：**
1. 建立名為 `nginx-deployment` 的 Deployment：
   - 映像：`nginx:1.24`
   - 副本數：3
   - 標籤：`app=nginx`
2. 將映像更新為 `nginx:1.25`
3. 檢視 rollout 狀態與歷史
4. 回滾到前一個版本
5. 擴展副本數到 5
6. 設定更新策略為 RollingUpdate，maxSurge=1，maxUnavailable=0

**驗證：**
```bash
oc rollout status deployment/nginx-deployment
oc rollout history deployment/nginx-deployment
```

---

### Exercise 2.2：ReplicaSet 與 Pod 關係

**題目：**
1. 建立一個 Deployment 並觀察自動建立的 ReplicaSet
2. 手動刪除一個 Pod，觀察 ReplicaSet 的行為
3. 直接擴展 ReplicaSet 的副本數（觀察會發生什麼）
4. 列出某個 Deployment 管理的所有 ReplicaSet
5. 檢視 ReplicaSet 的 ownerReferences

**驗證：**
```bash
oc get rs -l app=nginx
oc describe rs <rs-name> | grep -A5 "Controlled By"
```

---

### Exercise 2.3：DaemonSet

**題目：**
1. 建立一個 DaemonSet，在每個節點上執行 `fluentd` 日誌收集器
2. 使用 nodeSelector 限制 DaemonSet 只在特定節點執行
3. 檢視 DaemonSet 的狀態
4. 更新 DaemonSet 的映像版本

**YAML 模板：**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluentd:v1.16
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
```

---

### Exercise 2.4：StatefulSet

**題目：**
1. 建立一個 Headless Service
2. 建立一個 StatefulSet 部署 3 個 Redis 實例
3. 每個實例使用獨立的 PVC
4. 驗證 Pod 的命名規則（redis-0, redis-1, redis-2）
5. 測試有序啟動和關閉
6. 驗證 DNS 解析（redis-0.redis-headless.namespace.svc.cluster.local）

**驗證：**
```bash
oc get pods -l app=redis
oc get pvc
# 從另一個 Pod 測試 DNS
oc exec -it test-pod -- nslookup redis-0.redis-headless
```

---

### Exercise 2.5：Job 與 CronJob

**題目：**
1. 建立一個 Job，執行資料庫備份腳本（模擬）
2. 設定 Job 的 backoffLimit 為 3
3. 建立一個 CronJob，每 5 分鐘執行一次清理任務
4. 設定 CronJob 保留最近 3 個成功的 Job
5. 手動觸發 CronJob 執行
6. 暫停 CronJob

**YAML 模板：**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-backup
spec:
  template:
    spec:
      containers:
      - name: backup
        image: busybox
        command: ["sh", "-c", "echo 'Backing up database...' && sleep 10 && echo 'Done!'"]
      restartPolicy: Never
  backoffLimit: 3
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-job
spec:
  schedule: "*/5 * * * *"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: busybox
            command: ["sh", "-c", "echo 'Cleaning up...'"]
          restartPolicy: OnFailure
```

---

### Exercise 2.6：S2I 部署（OpenShift 特有）

**題目：**
1. 使用 S2I 從 GitHub 部署一個 Python 應用
2. 檢視 BuildConfig 和 Build 狀態
3. 追蹤建置日誌
4. 觸發新的建置
5. 設定 Webhook 自動觸發建置

**指令：**
```bash
oc new-app python:3.11~https://github.com/sclorg/django-ex.git --name=django-app
```

---

## Level 3：配置管理（Intermediate）

### Exercise 3.1：ConfigMap 操作

**題目：**
1. 從字面值建立 ConfigMap：
   ```
   database_host=postgres.default.svc
   database_port=5432
   log_level=INFO
   ```
2. 從檔案建立 ConfigMap（包含完整的設定檔）
3. 將 ConfigMap 掛載為環境變數
4. 將 ConfigMap 掛載為 Volume
5. 更新 ConfigMap 並驗證 Pod 是否自動更新

**驗證：**
```bash
oc exec <pod-name> -- env | grep database
oc exec <pod-name> -- cat /etc/config/application.properties
```

---

### Exercise 3.2：Secret 管理

**題目：**
1. 建立一個 Opaque Secret 包含資料庫認證
2. 建立一個 TLS Secret
3. 建立一個 Docker Registry Secret
4. 將 Secret 掛載為環境變數（只掛載特定 key）
5. 將 Secret 掛載為檔案
6. 使用 `stringData` 建立 Secret（比較與 `data` 的差異）

**驗證：**
```bash
# 確認 Secret 內容（base64 解碼）
oc get secret db-secret -o jsonpath='{.data.password}' | base64 -d
```

---

### Exercise 3.3：環境變數進階

**題目：**
建立一個 Pod，包含以下環境變數來源：
1. 靜態值
2. 從 ConfigMap 取得
3. 從 Secret 取得
4. 從 Pod 欄位取得（Downward API）
5. 從容器資源取得

**YAML 模板：**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-demo
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "env && sleep 3600"]
    env:
    - name: STATIC_VALUE
      value: "hello"
    - name: CONFIG_VALUE
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: log_level
    - name: SECRET_VALUE
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: CPU_LIMIT
      valueFrom:
        resourceFieldRef:
          containerName: app
          resource: limits.cpu
```

---

### Exercise 3.4：ConfigMap 熱更新

**題目：**
1. 建立一個 ConfigMap 並掛載到 Pod
2. 更新 ConfigMap 的內容
3. 驗證 Pod 內的檔案是否自動更新（注意：環境變數不會自動更新）
4. 實作應用程式配置熱載入機制

**提示：**
- Volume 掛載的 ConfigMap 會自動更新（有延遲）
- 環境變數需要重啟 Pod 才會更新

---

## Level 4：網路與服務（Intermediate）

### Exercise 4.1：Service 類型

**題目：**
1. 建立 ClusterIP Service，暴露 nginx Deployment
2. 建立 NodePort Service
3. 建立 Headless Service（clusterIP: None）
4. 測試不同 Service 類型的 DNS 解析
5. 使用 ExternalName Service 映射外部服務

**驗證：**
```bash
# 從另一個 Pod 測試
oc run test-pod --image=busybox -it --rm -- wget -qO- http://nginx-service
oc run test-pod --image=busybox -it --rm -- nslookup nginx-service
```

---

### Exercise 4.2：Route 配置（OpenShift 特有）

**題目：**
1. 建立基本的 HTTP Route
2. 建立 HTTPS Route（Edge Termination）
3. 建立 HTTPS Route（Passthrough Termination）
4. 建立 HTTPS Route（Re-encrypt Termination）
5. 設定自訂主機名稱
6. 設定路徑型路由（Path-based Routing）

**YAML 模板：**
```yaml
# Edge Termination
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: secure-route
spec:
  host: myapp.apps-crc.testing
  to:
    kind: Service
    name: myapp-service
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
---
# Path-based Routing
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: api-route
spec:
  host: myapp.apps-crc.testing
  path: /api
  to:
    kind: Service
    name: api-service
```

---

### Exercise 4.3：Ingress Controller

**題目：**
1. 建立 Ingress 資源
2. 設定多個主機（Host-based routing）
3. 設定路徑分流
4. 設定 TLS
5. 設定註解調整 Ingress 行為

**YAML 模板：**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - app1.example.com
    - app2.example.com
    secretName: tls-secret
  rules:
  - host: app1.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
  - host: app2.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

---

### Exercise 4.4：NetworkPolicy

**題目：**
1. 建立 NetworkPolicy 拒絕所有入站流量
2. 允許來自同一命名空間的流量
3. 允許來自特定標籤 Pod 的流量
4. 允許來自特定命名空間的流量
5. 限制出站流量只能存取特定服務
6. 允許來自 OpenShift Ingress Controller 的流量

**YAML 模板：**
```yaml
# Deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
# Allow from same namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
---
# Allow specific pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

---

### Exercise 4.5：服務發現與 DNS

**題目：**
1. 驗證 Kubernetes DNS 服務
2. 測試 Service DNS 解析格式
3. 測試 Pod DNS 解析
4. 測試 Headless Service DNS 解析
5. 使用 ExternalName 建立外部服務別名

**指令：**
```bash
# 進入測試 Pod
oc run dns-test --image=busybox -it --rm -- sh

# 在 Pod 內測試
nslookup kubernetes.default.svc.cluster.local
nslookup <service-name>.<namespace>.svc.cluster.local
cat /etc/resolv.conf
```

---

## Level 5：儲存管理（Intermediate）

### Exercise 5.1：PersistentVolume 與 PVC

**題目：**
1. 檢視可用的 StorageClass
2. 建立 PVC 請求 1Gi 儲存空間
3. 將 PVC 掛載到 Pod
4. 驗證資料持久化（刪除 Pod 後重建）
5. 擴展 PVC 容量（如果 StorageClass 支援）

**YAML 模板：**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: crc-csi-hostpath-provisioner
---
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo 'Hello' > /data/test.txt && sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data-pvc
```

---

### Exercise 5.2：Volume 類型

**題目：**
建立一個 Pod，使用以下不同類型的 Volume：
1. emptyDir
2. hostPath（需要特殊權限）
3. configMap
4. secret
5. downwardAPI
6. projected（組合多個來源）

**YAML 模板：**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-volume-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: cache
      mountPath: /cache
    - name: config
      mountPath: /etc/config
    - name: secrets
      mountPath: /etc/secrets
      readOnly: true
    - name: podinfo
      mountPath: /etc/podinfo
  volumes:
  - name: cache
    emptyDir:
      sizeLimit: 100Mi
  - name: config
    configMap:
      name: app-config
  - name: secrets
    secret:
      secretName: app-secret
  - name: podinfo
    downwardAPI:
      items:
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels
      - path: "annotations"
        fieldRef:
          fieldPath: metadata.annotations
```

---

### Exercise 5.3：StatefulSet 儲存

**題目：**
1. 建立帶有 volumeClaimTemplates 的 StatefulSet
2. 驗證每個 Pod 都有獨立的 PVC
3. 縮減 StatefulSet 副本數，觀察 PVC 行為
4. 擴展 StatefulSet，驗證 PVC 重新綁定

---

### Exercise 5.4：儲存快照（進階）

**題目：**
1. 檢查是否有 VolumeSnapshotClass
2. 建立 PVC 的快照
3. 從快照還原 PVC

**YAML 模板：**
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: data-snapshot
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass
  source:
    persistentVolumeClaimName: data-pvc
---
# Restore from snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-pvc
spec:
  dataSource:
    name: data-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

---

## Level 6：安全性（Advanced）

### Exercise 6.1：RBAC 配置

**題目：**
1. 建立 ServiceAccount
2. 建立 Role，授予對 pods 和 deployments 的讀取權限
3. 建立 RoleBinding，將 Role 綁定到 ServiceAccount
4. 建立 ClusterRole 和 ClusterRoleBinding
5. 測試權限（使用 `oc auth can-i`）

**YAML 模板：**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
---
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
---
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

**驗證：**
```bash
oc auth can-i get pods --as=system:serviceaccount:exercises:app-sa
oc auth can-i create pods --as=system:serviceaccount:exercises:app-sa
```

---

### Exercise 6.2：Security Context Constraints（OpenShift）

**題目：**
1. 列出所有可用的 SCC
2. 檢視 `restricted` SCC 的詳細設定
3. 建立一個需要以 root 執行的 Pod（會失敗）
4. 授予 ServiceAccount `anyuid` SCC
5. 重新部署 Pod

**指令：**
```bash
# 列出 SCC
oc get scc

# 檢視詳情
oc describe scc restricted

# 授予 SCC
oc adm policy add-scc-to-user anyuid -z my-sa

# 檢查 Pod 使用的 SCC
oc get pod <pod-name> -o yaml | grep scc
```

---

### Exercise 6.3：Pod Security Standards

**題目：**
1. 建立一個 Pod，設定 `runAsNonRoot: true`
2. 設定 `readOnlyRootFilesystem: true`
3. 設定 `allowPrivilegeEscalation: false`
4. 移除所有 Capabilities
5. 設定 seccompProfile

**YAML 模板：**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
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

---

### Exercise 6.4：Secret 加密

**題目：**
1. 建立包含敏感資料的 Secret
2. 驗證 Secret 以 base64 編碼儲存
3. 使用 Sealed Secrets 或 External Secrets 加密（進階）
4. 設定 Secret 自動輪換

---

### Exercise 6.5：網路安全

**題目：**
1. 實作零信任網路模型：
   - 預設拒絕所有流量
   - 只允許必要的服務間通訊
2. 建立分層 NetworkPolicy：
   - Frontend 只能被 Ingress 存取
   - Backend 只能被 Frontend 存取
   - Database 只能被 Backend 存取

**YAML 模板：**
```yaml
# Default deny all
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# Allow frontend from ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-ingress
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          network.openshift.io/policy-group: ingress
---
# Allow backend from frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-from-frontend
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
---
# Allow database from backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-db-from-backend
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
```

---

## Level 7：監控與日誌（Advanced）

### Exercise 7.1：Prometheus 監控

**題目：**
1. 存取 OpenShift 內建的 Prometheus
2. 執行 PromQL 查詢：
   - 取得所有 Pod 的 CPU 使用率
   - 取得記憶體使用量
   - 計算 HTTP 請求率
3. 建立 ServiceMonitor 監控自訂應用

**PromQL 範例：**
```promql
# CPU 使用率
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod)

# 記憶體使用量
sum(container_memory_usage_bytes{container!=""}) by (pod)

# HTTP 請求率
sum(rate(http_requests_total[5m])) by (service)
```

---

### Exercise 7.2：ServiceMonitor 配置

**題目：**
1. 部署一個暴露 Prometheus metrics 的應用
2. 建立 ServiceMonitor
3. 驗證 Prometheus 已經抓取指標
4. 建立 Grafana Dashboard

**YAML 模板：**
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

**題目：**
1. 建立 PrometheusRule 定義告警
2. 設定 Pod 重啟次數告警
3. 設定 CPU 使用率告警
4. 設定記憶體使用率告警
5. 配置 Alertmanager 通知

**YAML 模板：**
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
        summary: "Pod {{ $labels.pod }} has restarted more than 5 times in the last hour"
    
    - alert: HighCPUUsage
      expr: |
        sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod) > 0.8
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} CPU usage is above 80%"
    
    - alert: HighMemoryUsage
      expr: |
        sum(container_memory_usage_bytes{container!=""}) by (pod) 
        / 
        sum(container_spec_memory_limit_bytes{container!=""}) by (pod) > 0.9
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} memory usage is above 90%"
```

---

### Exercise 7.4：日誌收集

**題目：**
1. 檢視 Pod 日誌
2. 使用 stern 同時查看多個 Pod 日誌
3. 配置應用輸出 JSON 格式日誌
4. 使用 OpenShift Logging 查詢日誌

**指令：**
```bash
# 基本日誌
oc logs <pod-name>
oc logs -f <pod-name>
oc logs <pod-name> --since=1h
oc logs <pod-name> --tail=100

# 多容器 Pod
oc logs <pod-name> -c <container-name>

# 所有容器
oc logs <pod-name> --all-containers

# 使用 stern（需安裝）
stern <pod-prefix> -n <namespace>
```

---

### Exercise 7.5：Liveness 與 Readiness Probe

**題目：**
1. 建立帶有 HTTP liveness probe 的 Pod
2. 建立帶有 TCP readiness probe 的 Pod
3. 建立帶有 exec probe 的 Pod
4. 設定 startup probe
5. 調整 probe 參數（initialDelaySeconds, periodSeconds, failureThreshold）
6. 模擬 probe 失敗並觀察 Pod 行為

**YAML 模板：**
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
    startupProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 0
      periodSeconds: 5
      failureThreshold: 30
    livenessProbe:
      httpGet:
        path: /healthz
        port: 80
      initialDelaySeconds: 0
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 80
      initialDelaySeconds: 0
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
```

---

## Level 8：CI/CD Pipeline（Advanced）

### Exercise 8.1：Tekton Task 開發

**題目：**
1. 建立一個簡單的 Task 執行 shell 指令
2. 建立一個帶參數的 Task
3. 建立一個使用 Workspace 的 Task
4. 建立一個有 Results 輸出的 Task

**YAML 模板：**
```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: echo-task
spec:
  params:
  - name: message
    type: string
    default: "Hello, Tekton!"
  results:
  - name: timestamp
    description: The execution timestamp
  steps:
  - name: echo
    image: alpine
    script: |
      echo "$(params.message)"
      echo $(date +%s) | tee $(results.timestamp.path)
---
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: git-clone-custom
spec:
  params:
  - name: url
    type: string
  - name: revision
    type: string
    default: main
  workspaces:
  - name: output
  results:
  - name: commit
    description: The commit SHA
  steps:
  - name: clone
    image: alpine/git
    script: |
      git clone $(params.url) $(workspaces.output.path)/source
      cd $(workspaces.output.path)/source
      git checkout $(params.revision)
      git rev-parse HEAD | tr -d '\n' | tee $(results.commit.path)
```

---

### Exercise 8.2：Tekton Pipeline 開發

**題目：**
建立一個完整的 CI/CD Pipeline，包含：
1. Git Clone
2. Unit Test
3. Build Image
4. Push Image
5. Deploy to Dev
6. Integration Test
7. Deploy to Prod（手動審批）

**YAML 模板：**
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
  
  - name: test
    taskRef:
      name: maven
    runAfter:
    - clone
    params:
    - name: GOALS
      value: ["test"]
    workspaces:
    - name: source
      workspace: shared-workspace
  
  - name: build
    taskRef:
      name: maven
    runAfter:
    - test
    params:
    - name: GOALS
      value: ["package", "-DskipTests"]
    workspaces:
    - name: source
      workspace: shared-workspace
  
  - name: build-image
    taskRef:
      name: buildah
    runAfter:
    - build
    params:
    - name: IMAGE
      value: $(params.image-name):$(tasks.clone.results.commit)
    workspaces:
    - name: source
      workspace: shared-workspace
  
  - name: deploy-dev
    taskRef:
      name: kubernetes-actions
    runAfter:
    - build-image
    params:
    - name: script
      value: |
        kubectl set image deployment/myapp \
          app=$(params.image-name):$(tasks.clone.results.commit) \
          -n dev
  
  finally:
  - name: notify
    taskRef:
      name: send-notification
    params:
    - name: status
      value: "$(tasks.status)"
```

---

### Exercise 8.3：Tekton Triggers

**題目：**
1. 建立 EventListener 接收 GitHub Webhook
2. 建立 TriggerBinding 解析 Webhook payload
3. 建立 TriggerTemplate 產生 PipelineRun
4. 設定 GitHub Webhook
5. 測試 Push 事件觸發 Pipeline

**YAML 模板：**
```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: github-listener
spec:
  serviceAccountName: pipeline
  triggers:
  - name: github-push-trigger
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
  - name: git-repo-name
    value: $(body.repository.name)
---
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: pipeline-template
spec:
  params:
  - name: git-url
  - name: git-revision
  - name: git-repo-name
  resourcetemplates:
  - apiVersion: tekton.dev/v1
    kind: PipelineRun
    metadata:
      generateName: $(tt.params.git-repo-name)-
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

### Exercise 8.4：Pipeline 最佳實踐

**題目：**
1. 實作 Pipeline 快取機制（Maven cache, npm cache）
2. 設定 Pipeline 超時
3. 實作條件執行（When Expressions）
4. 實作並行任務
5. 使用 Finally tasks 進行清理

**YAML 模板：**
```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: advanced-pipeline
spec:
  tasks:
  # 並行執行
  - name: unit-test
    taskRef:
      name: maven-test
    runAfter:
    - clone
  
  - name: lint
    taskRef:
      name: lint
    runAfter:
    - clone
  
  - name: security-scan
    taskRef:
      name: trivy-scan
    runAfter:
    - clone
  
  # 條件執行
  - name: deploy-prod
    taskRef:
      name: deploy
    when:
    - input: "$(params.environment)"
      operator: in
      values: ["production"]
    runAfter:
    - build
  
  # Finally tasks
  finally:
  - name: cleanup
    taskRef:
      name: cleanup-workspace
  
  - name: report
    taskRef:
      name: send-report
    params:
    - name: aggregateTasksStatus
      value: "$(tasks.status)"
```

---

## Level 9：Operator 開發（Expert）

### Exercise 9.1：建立簡單 Operator

**題目：**
使用 Operator SDK 建立一個管理 Nginx 部署的 Operator：
1. 初始化 Operator 專案
2. 定義 CRD（NginxDeployment）
3. 實作 Reconcile 邏輯
4. 測試 Operator

**步驟：**
```bash
# 初始化專案
operator-sdk init --domain=example.com --repo=github.com/myorg/nginx-operator

# 建立 API
operator-sdk create api --group=web --version=v1 --kind=NginxDeployment --resource --controller

# 編輯 types
# 編輯 controller
# 生成 manifests
make manifests

# 本地測試
make install
make run
```

---

### Exercise 9.2：Operator Reconcile 邏輯

**題目：**
實作以下 Reconcile 邏輯：
1. 檢查 CR 是否存在
2. 建立或更新 Deployment
3. 建立或更新 Service
4. 更新 CR Status
5. 處理 Finalizer
6. 實作錯誤處理和重試

**Go 程式碼模板：**
```go
func (r *NginxDeploymentReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)

    // Fetch CR
    nginx := &webv1.NginxDeployment{}
    if err := r.Get(ctx, req.NamespacedName, nginx); err != nil {
        if errors.IsNotFound(err) {
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, err
    }

    // Check Finalizer
    if nginx.ObjectMeta.DeletionTimestamp.IsZero() {
        if !containsString(nginx.GetFinalizers(), finalizerName) {
            controllerutil.AddFinalizer(nginx, finalizerName)
            if err := r.Update(ctx, nginx); err != nil {
                return ctrl.Result{}, err
            }
        }
    } else {
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

    // Reconcile Deployment
    deployment := &appsv1.Deployment{}
    err := r.Get(ctx, types.NamespacedName{Name: nginx.Name, Namespace: nginx.Namespace}, deployment)
    if err != nil && errors.IsNotFound(err) {
        dep := r.deploymentForNginx(nginx)
        if err := r.Create(ctx, dep); err != nil {
            return ctrl.Result{}, err
        }
        return ctrl.Result{Requeue: true}, nil
    } else if err != nil {
        return ctrl.Result{}, err
    }

    // Update if needed
    if *deployment.Spec.Replicas != nginx.Spec.Replicas {
        deployment.Spec.Replicas = &nginx.Spec.Replicas
        if err := r.Update(ctx, deployment); err != nil {
            return ctrl.Result{}, err
        }
    }

    // Update Status
    nginx.Status.ReadyReplicas = deployment.Status.ReadyReplicas
    if err := r.Status().Update(ctx, nginx); err != nil {
        return ctrl.Result{}, err
    }

    return ctrl.Result{}, nil
}
```

---

### Exercise 9.3：Operator Lifecycle Manager (OLM)

**題目：**
1. 建立 ClusterServiceVersion (CSV)
2. 建立 Operator Bundle
3. 使用 OLM 安裝 Operator
4. 設定 Operator 升級策略

**指令：**
```bash
# 生成 bundle
make bundle

# 建置 bundle image
make bundle-build bundle-push

# 使用 OLM 安裝
operator-sdk run bundle quay.io/myorg/nginx-operator-bundle:v0.1.0
```

---

## Level 10：綜合情境題（Expert）

### Scenario 10.1：微服務架構部署

**情境：**
你需要部署一個電子商務平台，包含以下服務：
- Frontend (React)
- API Gateway (Spring Cloud Gateway)
- Product Service (Spring Boot)
- Order Service (Spring Boot)
- User Service (Spring Boot)
- PostgreSQL (Database)
- Redis (Cache)
- RabbitMQ (Message Queue)

**要求：**
1. 為每個服務建立 Deployment 和 Service
2. 配置 ConfigMap 和 Secret
3. 設定健康檢查
4. 配置 HPA 自動擴展
5. 設定 NetworkPolicy
6. 配置 Ingress/Route
7. 建立 CI/CD Pipeline

---

### Scenario 10.2：災難恢復演練

**情境：**
模擬以下故障場景並實施恢復：
1. Pod 被意外刪除
2. Deployment 配置錯誤導致所有 Pod crash
3. PVC 資料損壞（使用 VolumeSnapshot 恢復）
4. 整個 Namespace 被刪除
5. 節點故障（模擬）

**要求：**
1. 文檔化每個故障的恢復步驟
2. 建立自動化恢復腳本
3. 設定告警

---

### Scenario 10.3：藍綠部署實作

**情境：**
實作一個藍綠部署策略：
1. 部署 v1 版本（Blue）
2. 部署 v2 版本（Green）
3. 切換流量到 Green
4. 驗證 Green 正常運作
5. 如有問題，快速回滾到 Blue

**要求：**
1. 使用獨立的 Deployment
2. 使用 Service 和 Route 控制流量
3. 建立切換腳本
4. 建立健康檢查和自動回滾機制

---

### Scenario 10.4：多租戶隔離

**情境：**
設計一個多租戶架構，支援三個團隊：
- Team A: Frontend 開發
- Team B: Backend 開發
- Team C: Database 管理

**要求：**
1. 為每個團隊建立獨立的 Namespace
2. 設定 RBAC，限制各團隊只能存取自己的資源
3. 設定 ResourceQuota 限制資源使用
4. 設定 NetworkPolicy 隔離網路
5. 設定共享服務的存取權限

---

### Scenario 10.5：效能調優

**情境：**
一個應用在高流量下效能不佳，需要進行調優。

**要求：**
1. 分析 Prometheus 指標找出瓶頸
2. 調整 Pod 資源配置
3. 配置 HPA
4. 優化 JVM 設定（如果是 Java 應用）
5. 配置 PodDisruptionBudget
6. 實施 Pod Anti-Affinity

---

## 模擬考題：EX280 風格

### Exam Task 1：專案與使用者管理（5 分鐘）

**任務：**
1. 以 `kubeadmin` 登入叢集
2. 建立專案 `project-alpha`
3. 建立使用者 `developer1`（使用 HTPasswd Identity Provider）
4. 授予 `developer1` 對 `project-alpha` 的 `edit` 角色

---

### Exam Task 2：部署應用（10 分鐘）

**任務：**
1. 在 `project-alpha` 專案中，使用 S2I 部署應用：
   - Git Repo: https://github.com/sclorg/nodejs-ex.git
   - 名稱: `nodejs-app`
2. 確保應用有 3 個副本
3. 建立 Route 暴露應用，主機名為 `nodejs.apps-crc.testing`
4. 設定 TLS（Edge Termination）

---

### Exam Task 3：配置管理（10 分鐘）

**任務：**
1. 建立 ConfigMap `app-config`，包含：
   ```
   DB_HOST=postgres.project-alpha.svc
   DB_PORT=5432
   LOG_LEVEL=INFO
   ```
2. 建立 Secret `db-credentials`，包含：
   ```
   DB_USER=appuser
   DB_PASSWORD=secretpass
   ```
3. 修改 `nodejs-app` Deployment，將 ConfigMap 和 Secret 作為環境變數注入

---

### Exam Task 4：資源限制（5 分鐘）

**任務：**
1. 為 `project-alpha` 設定 ResourceQuota：
   - CPU requests: 2
   - Memory requests: 4Gi
   - Pods: 10
2. 設定 LimitRange：
   - 預設 CPU limit: 500m
   - 預設 Memory limit: 512Mi

---

### Exam Task 5：網路政策（10 分鐘）

**任務：**
1. 建立 NetworkPolicy，只允許來自 `project-alpha` 內部的流量
2. 允許來自 OpenShift Ingress Controller 的流量
3. 拒絕所有其他入站流量

---

### Exam Task 6：故障排除（15 分鐘）

**任務：**
一個名為 `broken-app` 的 Deployment 無法正常執行。請診斷並修復問題。

可能的問題包括：
- 映像不存在
- 資源配置不足
- 健康檢查失敗
- 配置錯誤
- 權限問題

---

### Exam Task 7：自動擴展（5 分鐘）

**任務：**
為 `nodejs-app` 配置 HorizontalPodAutoscaler：
- 最小副本: 2
- 最大副本: 10
- 目標 CPU 使用率: 70%

---

### Exam Task 8：持久化儲存（10 分鐘）

**任務：**
1. 建立 PVC `data-storage`，大小 5Gi
2. 建立一個 Pod 使用該 PVC
3. 驗證資料在 Pod 重啟後仍然存在

---

## 解答與提示

### Exercise 1.1 解答

```bash
# 1. 建立專案
oc new-project dev-team-a --description="Development Team A Project" --display-name="Dev Team A"

# 2. 建立專案
oc new-project dev-team-b

# 3. 列出專案
oc get projects

# 4. 切換專案
oc project dev-team-a

# 5. 新增標籤
oc label namespace dev-team-a environment=development

# 6. 刪除專案
oc delete project dev-team-b
```

---

### Exercise 1.2 解答

```bash
# 1. 建立 Pod
oc run web-pod --image=nginx:1.25

# 2. 檢視詳細資訊
oc describe pod web-pod

# 3. 檢視日誌
oc logs web-pod

# 4. 進入 Pod
oc exec web-pod -- nginx -v

# 5. 端口轉發
oc port-forward web-pod 8080:80

# 6. 刪除 Pod
oc delete pod web-pod
```

---

### Exercise 1.3 解答

```yaml
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

---

### Exercise 2.1 解答

```bash
# 1. 建立 Deployment
oc create deployment nginx-deployment --image=nginx:1.24 --replicas=3

# 2. 更新映像
oc set image deployment/nginx-deployment nginx=nginx:1.25

# 3. 檢視狀態與歷史
oc rollout status deployment/nginx-deployment
oc rollout history deployment/nginx-deployment

# 4. 回滾
oc rollout undo deployment/nginx-deployment

# 5. 擴展
oc scale deployment/nginx-deployment --replicas=5

# 6. 設定更新策略
oc patch deployment nginx-deployment -p '{"spec":{"strategy":{"type":"RollingUpdate","rollingUpdate":{"maxSurge":1,"maxUnavailable":0}}}}'
```

---

### Exercise 4.4 解答（NetworkPolicy）

```yaml
# 完整的分層 NetworkPolicy
---
# Default deny all
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: my-app
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# Allow DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: my-app
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
---
# Allow frontend from ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-frontend
  namespace: my-app
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          network.openshift.io/policy-group: ingress
---
# Allow backend from frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: my-app
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
---
# Frontend can reach backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-egress-to-backend
  namespace: my-app
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 8080
```

---

### Exam Task 2 解答

```bash
# 切換專案
oc project project-alpha

# S2I 部署
oc new-app nodejs:18~https://github.com/sclorg/nodejs-ex.git --name=nodejs-app

# 等待建置完成
oc logs -f buildconfig/nodejs-app

# 擴展到 3 副本
oc scale deployment/nodejs-app --replicas=3

# 建立 TLS Route
oc create route edge nodejs-route \
  --service=nodejs-app \
  --hostname=nodejs.apps-crc.testing \
  --insecure-policy=Redirect
```

---

### Exam Task 3 解答

```bash
# 建立 ConfigMap
oc create configmap app-config \
  --from-literal=DB_HOST=postgres.project-alpha.svc \
  --from-literal=DB_PORT=5432 \
  --from-literal=LOG_LEVEL=INFO

# 建立 Secret
oc create secret generic db-credentials \
  --from-literal=DB_USER=appuser \
  --from-literal=DB_PASSWORD=secretpass

# 注入環境變數
oc set env deployment/nodejs-app --from=configmap/app-config
oc set env deployment/nodejs-app --from=secret/db-credentials
```

---

### 常用除錯指令速查

```bash
# Pod 狀態
oc get pods -o wide
oc describe pod <pod-name>
oc logs <pod-name> --previous

# 事件
oc get events --sort-by='.lastTimestamp'

# 資源使用
oc adm top pods
oc adm top nodes

# 進入容器
oc exec -it <pod-name> -- /bin/bash
oc rsh <pod-name>

# 端口轉發
oc port-forward <pod-name> 8080:80

# 除錯節點
oc debug node/<node-name>

# 檢查權限
oc auth can-i <verb> <resource>
oc auth can-i --list

# 檢查 API 資源
oc api-resources
oc explain pod.spec.containers
```

---

## 附錄：練習進度追蹤表

| Level | 練習編號 | 題目 | 完成日期 | 備註 |
|-------|---------|------|---------|------|
| 1 | 1.1 | 專案管理 | | |
| 1 | 1.2 | Pod 操作 | | |
| 1 | 1.3 | YAML 建立 | | |
| 1 | 1.4 | 標籤選擇器 | | |
| 1 | 1.5 | 輸出格式 | | |
| 2 | 2.1 | Deployment | | |
| 2 | 2.2 | ReplicaSet | | |
| 2 | 2.3 | DaemonSet | | |
| 2 | 2.4 | StatefulSet | | |
| 2 | 2.5 | Job/CronJob | | |
| 2 | 2.6 | S2I | | |
| 3 | 3.1 | ConfigMap | | |
| 3 | 3.2 | Secret | | |
| 3 | 3.3 | 環境變數 | | |
| 3 | 3.4 | 熱更新 | | |
| 4 | 4.1 | Service | | |
| 4 | 4.2 | Route | | |
| 4 | 4.3 | Ingress | | |
| 4 | 4.4 | NetworkPolicy | | |
| 4 | 4.5 | DNS | | |
| 5 | 5.1 | PV/PVC | | |
| 5 | 5.2 | Volume 類型 | | |
| 5 | 5.3 | StatefulSet 儲存 | | |
| 5 | 5.4 | 快照 | | |
| 6 | 6.1 | RBAC | | |
| 6 | 6.2 | SCC | | |
| 6 | 6.3 | Pod Security | | |
| 6 | 6.4 | Secret 加密 | | |
| 6 | 6.5 | 網路安全 | | |
| 7 | 7.1 | Prometheus | | |
| 7 | 7.2 | ServiceMonitor | | |
| 7 | 7.3 | 告警規則 | | |
| 7 | 7.4 | 日誌 | | |
| 7 | 7.5 | Probe | | |
| 8 | 8.1 | Tekton Task | | |
| 8 | 8.2 | Tekton Pipeline | | |
| 8 | 8.3 | Triggers | | |
| 8 | 8.4 | 最佳實踐 | | |
| 9 | 9.1 | Operator 建立 | | |
| 9 | 9.2 | Reconcile | | |
| 9 | 9.3 | OLM | | |
| 10 | 10.1 | 微服務 | | |
| 10 | 10.2 | 災難恢復 | | |
| 10 | 10.3 | 藍綠部署 | | |
| 10 | 10.4 | 多租戶 | | |
| 10 | 10.5 | 效能調優 | | |

---

*文件版本：1.0*  
*練習題數：50+*  
*預估完成時間：40-60 小時*
