# OpenShift/Kubernetes Level 5 儲存管理實作指南

> 基於 CRC (CodeReady Containers) 環境的實戰練習結果

---

## Exercise 5.1：PersistentVolume 與 PVC

### 題目
1. 檢視可用的 StorageClass
2. 建立 PVC 請求儲存空間
3. 將 PVC 掛載到 Pod
4. 驗證資料持久化

### 指令與結果

#### 1. 檢視 StorageClass

```bash
oc get storageclass
```

**輸出：**
```
NAME                                     PROVISIONER                        RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
crc-csi-hostpath-provisioner (default)   kubevirt.io.hostpath-provisioner   Retain          WaitForFirstConsumer   false                  68d
```

#### 2. 建立 PVC

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
```

```bash
oc apply -f data-pvc.yaml
oc get pvc data-pvc
```

**輸出：**
```
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                   AGE
data-pvc   Bound    pvc-541b2d61-5028-43ab-992f-ce27d066ae42   30Gi       RWO            crc-csi-hostpath-provisioner   52s
```

#### 3. 掛載 PVC 到 Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo 'Data persisted' > /data/test.txt && sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data-pvc
```

#### 4. 驗證資料持久化

```bash
# 寫入資料
oc exec pvc-pod -- cat /data/test.txt

# 刪除 Pod
oc delete pod pvc-pod

# 重建 Pod
oc apply -f pvc-pod.yaml

# 驗證資料仍存在
oc exec pvc-pod -- cat /data/test.txt
```

**結果：** 資料在 Pod 刪除重建後仍然保留

---

## Exercise 5.2：Volume 類型

### 題目
建立 Pod 使用多種 Volume 類型：
1. emptyDir
2. configMap
3. secret
4. downwardAPI
5. projected（組合多來源）

### YAML 定義

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-volume-pod
  labels:
    app: volume-demo
  annotations:
    description: "Demo pod"
spec:
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    # 1. emptyDir - 暫存空間
    - name: cache
      mountPath: /cache
    # 2. configMap
    - name: config
      mountPath: /etc/config
    # 3. secret
    - name: secrets
      mountPath: /etc/secrets
      readOnly: true
    # 4. downwardAPI
    - name: podinfo
      mountPath: /etc/podinfo
  volumes:
  # 1. emptyDir
  - name: cache
    emptyDir:
      sizeLimit: 100Mi
  # 2. configMap
  - name: config
    configMap:
      name: app-config
  # 3. secret
  - name: secrets
    secret:
      secretName: app-secret
  # 4. downwardAPI
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

### Projected Volume（組合多來源）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: projected-volume-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: all-in-one
      mountPath: /etc/projected
  volumes:
  - name: all-in-one
    projected:
      sources:
      - configMap:
          name: app-config
      - secret:
          name: app-secret
      - downwardAPI:
          items:
          - path: "podname"
            fieldRef:
              fieldPath: metadata.name
```

---

## Exercise 5.3：StatefulSet 儲存

### 題目
1. 建立帶有 volumeClaimTemplates 的 StatefulSet
2. 驗證每個 Pod 有獨立的 PVC
3. 縮減/擴展副本數，觀察 PVC 行為

### YAML 定義

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  ports:
  - port: 3306
  clusterIP: None
  selector:
    app: mysql
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql-headless"
  replicas: 2
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

### 執行結果

```bash
# Pods 依序建立
oc get pods -l app=mysql
```

**輸出：**
```
NAME      READY   STATUS    RESTARTS   AGE
mysql-0   1/1     Running   0          15s
mysql-1   1/1     Running   0          8s
```

```bash
# PVCs 自動建立（命名格式：<template-name>-<statefulset>-<ordinal>）
oc get pvc -l app=mysql
```

**輸出：**
```
NAME                 STATUS   VOLUME     CAPACITY   ACCESS MODES   AGE
mysql-data-mysql-0   Bound    pvc-xxx    30Gi       RWO            16s
mysql-data-mysql-1   Bound    pvc-yyy    30Gi       RWO            10s
```

### 重要行為

| 操作 | PVC 行為 |
|------|---------|
| 縮減副本數 | PVC 保留（不刪除） |
| 擴展副本數 | 自動重新綁定已存在的 PVC |
| 刪除 StatefulSet | 依據 `persistentVolumeClaimRetentionPolicy` |

---

## Exercise 5.4：儲存快照（進階）

### 前提條件

VolumeSnapshot 需要：
- CSI 驅動程式支援快照
- 已配置 VolumeSnapshotClass
- snapshot.storage.k8s.io API 可用

### 建立快照

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: data-snapshot
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass
  source:
    persistentVolumeClaimName: data-pvc
```

### 從快照還原

```yaml
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

**注意：** CRC 環境可能不支援 VolumeSnapshot，需要額外配置 CSI 驅動。

---

## 常用指令速查表

### PVC 操作

| 指令 | 說明 |
|------|------|
| `oc get pvc` | 列出 PVC |
| `oc get pv` | 列出 PV |
| `oc describe pvc <name>` | 檢視 PVC 詳情 |
| `oc delete pvc <name>` | 刪除 PVC |

### StorageClass 操作

| 指令 | 說明 |
|------|------|
| `oc get storageclass` | 列出 StorageClass |
| `oc describe storageclass <name>` | 檢視詳情 |

### Volume 檢視

```bash
# 檢視 Pod 的 Volume 掛載
oc describe pod <pod> | grep -A20 "Volumes:"

# 檢視 Pod 內的掛載點
oc exec <pod> -- df -h

# 列出掛載的檔案
oc exec <pod> -- ls -la /data
```

---

## Volume 類型比較

| 類型 | 持久化 | 共享 | 使用場景 |
|------|--------|------|----------|
| emptyDir | ✗ | Pod 內容器共享 | 暫存、快取 |
| hostPath | ✓ | 節點共享 | 開發測試 |
| configMap | ✗ | ✗ | 配置檔案 |
| secret | ✗ | ✗ | 敏感資料 |
| PVC | ✓ | 依 accessMode | 持久化資料 |
| downwardAPI | ✗ | ✗ | Pod 資訊注入 |
| projected | ✗ | ✗ | 組合多來源 |

---

## Access Modes 說明

| Mode | 縮寫 | 說明 |
|------|------|------|
| ReadWriteOnce | RWO | 單一節點讀寫 |
| ReadOnlyMany | ROX | 多節點唯讀 |
| ReadWriteMany | RWX | 多節點讀寫 |
| ReadWriteOncePod | RWOP | 單一 Pod 讀寫（K8s 1.22+） |

---

## Reclaim Policy 說明

| Policy | 說明 |
|--------|------|
| Retain | PV 保留，需手動清理 |
| Delete | PV 自動刪除 |
| Recycle | 清空資料後重用（已棄用） |

---

*文件產生時間：2026-02-04*
*測試環境：CRC 2.57.0 / OpenShift 4.x*
