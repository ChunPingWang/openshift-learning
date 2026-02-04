# OpenShift/Kubernetes Level 2 應用部署實作指南

> 基於 CRC (CodeReady Containers) 環境的實戰練習結果

> **⚠️ 重要提醒：** `kubeadmin` 的密碼在每次 CRC 安裝時都會自動產生，每個環境的密碼都不同。請使用 `crc console --credentials` 指令取得您環境的實際密碼。

---

## Exercise 2.1：Deployment 管理

### 題目
1. 建立 Deployment（nginx:1.24，3 副本）
2. 更新映像至 nginx:1.25
3. 檢視 rollout 狀態與歷史
4. 回滾到前一個版本
5. 擴展副本數到 5
6. 設定更新策略

### 指令與結果

#### 1. 建立 Deployment

```bash
oc create deployment nginx-deployment --image=nginx:1.24 --replicas=3
```

**輸出：**
```
deployment.apps/nginx-deployment created
```

檢視狀態：
```bash
oc get deployment nginx-deployment
oc get pods -l app=nginx-deployment
```

**輸出：**
```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           30s

NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-64bc7ffccb-8nc4t   1/1     Running   0          30s
nginx-deployment-64bc7ffccb-b5h2j   1/1     Running   0          30s
nginx-deployment-64bc7ffccb-p7bpf   1/1     Running   0          30s
```

#### 2. 更新映像

```bash
oc set image deployment/nginx-deployment nginx=nginx:1.25 --record
```

**輸出：**
```
deployment.apps/nginx-deployment image updated
```

等待 rollout 完成：
```bash
oc rollout status deployment/nginx-deployment
```

#### 3. 檢視 Rollout 歷史

```bash
oc rollout history deployment/nginx-deployment
```

**輸出：**
```
REVISION  CHANGE-CAUSE
1         <none>
2         oc set image deployment/nginx-deployment nginx=nginx:1.25 --record=true
```

#### 4. 回滾到前一個版本

```bash
oc rollout undo deployment/nginx-deployment
```

**輸出：**
```
deployment.apps/nginx-deployment rolled back
```

驗證回滾後的映像：
```bash
oc get deployment nginx-deployment -o jsonpath='{.spec.template.spec.containers[0].image}'
```

**輸出：**
```
nginx:1.24
```

#### 5. 擴展副本數

```bash
oc scale deployment/nginx-deployment --replicas=5
```

**輸出：**
```
deployment.apps/nginx-deployment scaled
```

#### 6. 設定更新策略

```bash
oc patch deployment nginx-deployment -p '{"spec":{"strategy":{"type":"RollingUpdate","rollingUpdate":{"maxSurge":1,"maxUnavailable":0}}}}'
```

**驗證策略設定：**
```bash
oc get deployment nginx-deployment -o jsonpath='{.spec.strategy}' | jq .
```

**輸出：**
```json
{
  "rollingUpdate": {
    "maxSurge": 1,
    "maxUnavailable": 0
  },
  "type": "RollingUpdate"
}
```

---

## Exercise 2.2：ReplicaSet 與 Pod 關係

### 題目
1. 觀察 Deployment 自動建立的 ReplicaSet
2. 手動刪除 Pod，觀察 ReplicaSet 行為
3. 直接擴展 ReplicaSet（觀察結果）
4. 檢視 ownerReferences

### 指令與結果

#### 1. 觀察 ReplicaSet

```bash
oc get rs -l app=nginx-deployment
```

**輸出：**
```
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-64bc7ffccb   5         5         5       5m
nginx-deployment-6f4b6b6dfc   0         0         0       4m
```

#### 2. 刪除 Pod 後觀察

```bash
# 刪除一個 Pod
POD_NAME=$(oc get pods -l app=nginx-deployment -o jsonpath='{.items[0].metadata.name}')
oc delete pod $POD_NAME

# 觀察 ReplicaSet 自動重建
oc get pods -l app=nginx-deployment
```

**重點：** ReplicaSet 會自動建立新的 Pod 以維持期望的副本數。

#### 3. 直接擴展 ReplicaSet

```bash
RS_NAME=$(oc get rs -l app=nginx-deployment -o jsonpath='{.items[0].metadata.name}')
oc scale rs/$RS_NAME --replicas=2
```

**重點：** Deployment 控制器會將 ReplicaSet 恢復到 Deployment 定義的副本數！

#### 4. 檢視 ownerReferences

```bash
oc get rs -l app=nginx-deployment -o jsonpath='{.items[0].metadata.ownerReferences}' | jq .
```

**輸出：**
```json
[
  {
    "apiVersion": "apps/v1",
    "blockOwnerDeletion": true,
    "controller": true,
    "kind": "Deployment",
    "name": "nginx-deployment",
    "uid": "b0260451-b00a-44ca-bc88-f95d38c4e519"
  }
]
```

---

## Exercise 2.3：DaemonSet

### 題目
1. 建立 DaemonSet 在每個節點執行 fluentd
2. 使用 nodeSelector 限制執行節點
3. 更新 DaemonSet 映像

### YAML 定義

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  labels:
    app: fluentd
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
        image: fluent/fluentd:v1.16-debian-1
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
```

### 指令與結果

```bash
# 建立 DaemonSet
oc apply -f fluentd-daemonset.yaml

# 檢視狀態
oc get daemonset fluentd
```

**輸出：**
```
NAME      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
fluentd   1         1         1       1            1           <none>          30s
```

#### 更新 DaemonSet 映像

```bash
oc set image daemonset/fluentd fluentd=fluent/fluentd:v1.17-debian-1
oc rollout status daemonset/fluentd
```

---

## Exercise 2.4：StatefulSet

### 題目
1. 建立 Headless Service
2. 建立 StatefulSet 部署 Redis
3. 驗證 Pod 命名規則
4. 驗證 PVC 建立

### YAML 定義

#### Headless Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-headless
  labels:
    app: redis
spec:
  ports:
  - port: 6379
    name: redis
  clusterIP: None
  selector:
    app: redis
```

#### StatefulSet
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: "redis-headless"
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
          name: redis
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Mi
```

### 指令與結果

```bash
# 建立資源
oc apply -f redis-headless-svc.yaml
oc apply -f redis-statefulset.yaml

# 檢視 Pod（注意命名規則）
oc get pods -l app=redis
```

**輸出：**
```
NAME      READY   STATUS    RESTARTS   AGE
redis-0   1/1     Running   0          25s
redis-1   1/1     Running   0          13s
redis-2   1/1     Running   0          11s
```

**重點：** StatefulSet 的 Pod 按照 `<statefulset-name>-<ordinal>` 格式命名，且按順序建立。

```bash
# 檢視 PVC
oc get pvc
```

**輸出：**
```
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                   AGE
data-redis-0   Bound    pvc-7062b064-f044-41e2-84bf-fa5aef32fe62   30Gi       RWO            crc-csi-hostpath-provisioner   25s
data-redis-1   Bound    pvc-1a50c4b8-2adb-4fc0-859d-8559cd785aac   30Gi       RWO            crc-csi-hostpath-provisioner   13s
data-redis-2   Bound    pvc-709febdf-5807-420a-a14e-603e3245e5a8   30Gi       RWO            crc-csi-hostpath-provisioner   11s
```

#### 測試 DNS 解析

```bash
# 從另一個 Pod 測試 DNS
oc run dns-test --image=busybox -it --rm -- nslookup redis-0.redis-headless
```

---

## Exercise 2.5：Job 與 CronJob

### 題目
1. 建立 Job 執行備份任務
2. 設定 backoffLimit
3. 建立 CronJob
4. 手動觸發 CronJob
5. 暫停 CronJob

### YAML 定義

#### Job
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
        command: ["sh", "-c", "echo 'Starting backup...' && sleep 5 && echo 'Backup completed!' && date"]
      restartPolicy: Never
  backoffLimit: 3
```

#### CronJob
```yaml
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
            command: ["sh", "-c", "echo 'Cleanup task running at:' && date"]
          restartPolicy: OnFailure
```

### 指令與結果

```bash
# 建立 Job
oc apply -f db-backup-job.yaml

# 等待完成
oc wait --for=condition=complete job/db-backup --timeout=60s

# 檢視日誌
oc logs job/db-backup
```

**輸出：**
```
Starting backup...
Backup completed!
Wed Feb  4 11:55:09 UTC 2026
```

```bash
# 建立 CronJob
oc apply -f cleanup-cronjob.yaml

# 手動觸發
oc create job cleanup-manual --from=cronjob/cleanup-job

# 暫停 CronJob
oc patch cronjob cleanup-job -p '{"spec":{"suspend":true}}'
```

**驗證 CronJob 狀態：**
```
NAME          SCHEDULE      TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cleanup-job   */5 * * * *   <none>     True      0        <none>          7s
```

---

## Exercise 2.6：S2I 部署（OpenShift 特有）

### 題目
1. 使用 S2I 從 GitHub 部署 Python 應用
2. 檢視 BuildConfig 和 Build 狀態
3. 追蹤建置日誌
4. 觸發新的建置

### 指令與結果

```bash
# 使用 S2I 部署
oc new-app python:3.11-ubi9~https://github.com/sclorg/django-ex.git --name=django-app --strategy=source
```

**輸出：**
```
--> Found image 4b5a19e in image stream "openshift/python" under tag "3.11-ubi9"

    Python 3.11
    -----------
    Python 3.11 available as container is a base platform for building and running
    various Python 3.11 applications and frameworks.

--> Creating resources ...
    imagestream.image.openshift.io "django-app" created
    buildconfig.build.openshift.io "django-app" created
    deployment.apps "django-app" created
    service "django-app" created
--> Success
```

```bash
# 檢視 BuildConfig
oc get buildconfig django-app

# 檢視 Build 狀態
oc get builds

# 追蹤建置日誌
oc logs -f buildconfig/django-app

# 建立 Route
oc expose service/django-app

# 觸發新建置
oc start-build django-app
```

**注意：** 在 CRC 環境中，由於資源限制，S2I 建置可能會因為 Pod 被驅逐而失敗。在生產環境中通常不會有此問題。

---

## 常用指令速查表

### Deployment 操作

| 指令 | 說明 |
|------|------|
| `oc create deployment <name> --image=<image>` | 建立 Deployment |
| `oc set image deployment/<name> <container>=<image>` | 更新映像 |
| `oc rollout status deployment/<name>` | 檢視 rollout 狀態 |
| `oc rollout history deployment/<name>` | 檢視 rollout 歷史 |
| `oc rollout undo deployment/<name>` | 回滾到前一版本 |
| `oc scale deployment/<name> --replicas=<n>` | 調整副本數 |

### StatefulSet 操作

| 指令 | 說明 |
|------|------|
| `oc get statefulset` | 列出 StatefulSet |
| `oc scale statefulset/<name> --replicas=<n>` | 調整副本數 |
| `oc delete statefulset <name> --cascade=orphan` | 刪除但保留 Pod |

### Job/CronJob 操作

| 指令 | 說明 |
|------|------|
| `oc get jobs` | 列出 Jobs |
| `oc get cronjobs` | 列出 CronJobs |
| `oc create job <name> --from=cronjob/<cron>` | 手動觸發 CronJob |
| `oc logs job/<name>` | 檢視 Job 日誌 |

### S2I 操作

| 指令 | 說明 |
|------|------|
| `oc new-app <builder>~<git-url>` | S2I 部署 |
| `oc get builds` | 列出建置 |
| `oc logs -f buildconfig/<name>` | 追蹤建置日誌 |
| `oc start-build <name>` | 觸發新建置 |

---

*文件產生時間：2026-02-04*
*測試環境：CRC 2.57.0 / OpenShift 4.x*
