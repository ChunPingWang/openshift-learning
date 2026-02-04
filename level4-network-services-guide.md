# OpenShift/Kubernetes Level 4 網路與服務實作指南

> 基於 CRC (CodeReady Containers) 環境的實戰練習結果

---

## Exercise 4.1：Service 類型

### 題目
1. 建立 ClusterIP Service
2. 建立 NodePort Service
3. 建立 Headless Service
4. 建立 ExternalName Service
5. 測試 DNS 解析

### 指令與結果

#### 1. ClusterIP Service（預設類型）

```bash
oc expose deployment nginx-svc-test --name=nginx-clusterip --port=80 --target-port=80 --type=ClusterIP
```

**輸出：**
```
NAME              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
nginx-clusterip   ClusterIP   10.217.4.131   <none>        80/TCP    0s
```

**特點：** 只能從叢集內部存取

#### 2. NodePort Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx-svc-test
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

**輸出：**
```
NAME             TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
nginx-nodeport   NodePort   10.217.5.37   <none>        80:30080/TCP   0s
```

**特點：** 可透過 `<NodeIP>:30080` 從外部存取

#### 3. Headless Service（clusterIP: None）

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
spec:
  clusterIP: None
  selector:
    app: nginx-svc-test
  ports:
  - port: 80
    targetPort: 80
```

**輸出：**
```
NAME             TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
nginx-headless   ClusterIP   None         <none>        80/TCP    0s
```

**特點：** DNS 查詢直接返回 Pod IP，適用於 StatefulSet

#### 4. ExternalName Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-api
spec:
  type: ExternalName
  externalName: api.example.com
```

**輸出：**
```
NAME           TYPE           CLUSTER-IP   EXTERNAL-IP       PORT(S)   AGE
external-api   ExternalName   <none>       api.example.com   <none>    0s
```

**特點：** 提供叢集內部的 DNS 別名指向外部服務

---

## Exercise 4.2：Route 配置（OpenShift 特有）

### 題目
1. 建立基本 HTTP Route
2. 建立 HTTPS Route（Edge Termination）
3. 建立 HTTPS Route（Passthrough Termination）
4. 建立路徑型路由

### 指令與結果

#### 1. 基本 HTTP Route

```bash
oc expose svc/nginx-clusterip --name=nginx-http-route
```

**輸出：**
```
NAME               HOST/PORT                                     PATH   SERVICES          PORT   TERMINATION   WILDCARD
nginx-http-route   nginx-http-route-exercises.apps-crc.testing          nginx-clusterip   80                   None
```

#### 2. HTTPS Route - Edge Termination

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: nginx-edge-route
spec:
  host: nginx-edge.apps-crc.testing
  to:
    kind: Service
    name: nginx-clusterip
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

**TLS 終止模式說明：**
- `edge`: TLS 在 Router 終止，後端為 HTTP
- `passthrough`: TLS 直接傳遞到 Pod，後端處理加密
- `reencrypt`: Router 終止後重新加密連接到後端

#### 3. 路徑型路由（Path-based Routing）

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: nginx-path-route
spec:
  host: myapp.apps-crc.testing
  path: /api
  to:
    kind: Service
    name: nginx-clusterip
```

---

## Exercise 4.3：Ingress Controller

### 題目
建立 Ingress 資源，設定多主機和路徑分流

### YAML 定義

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
  annotations:
    route.openshift.io/termination: edge
spec:
  rules:
  - host: app1.apps-crc.testing
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-clusterip
            port:
              number: 80
  - host: app2.apps-crc.testing
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: nginx-clusterip
            port:
              number: 80
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: nginx-clusterip
            port:
              number: 80
```

**重點：** OpenShift 會自動將 Ingress 轉換為 Route 資源

---

## Exercise 4.4：NetworkPolicy

### 題目
1. 預設拒絕所有入站流量
2. 允許同一命名空間內的流量
3. 實作分層網路策略（Frontend → Backend → Database）

### 指令與結果

#### 1. 預設拒絕所有入站流量

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

#### 2. 允許同一命名空間內的流量

```yaml
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
```

#### 3. 允許 Frontend 存取 Backend

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
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
```

#### 4. 允許 Backend 存取 Database

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
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

#### 5. 允許 OpenShift Ingress 流量

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-openshift-ingress
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
```

---

## Exercise 4.5：服務發現與 DNS

### 題目
1. 驗證 Kubernetes DNS 服務
2. 測試各種 DNS 解析格式
3. 檢視 Pod 的 DNS 配置

### 指令與結果

#### DNS 服務

```bash
oc get svc -n openshift-dns
```

**輸出：**
```
NAME          TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                  AGE
dns-default   ClusterIP   10.217.4.10   <none>        53/UDP,53/TCP,9154/TCP   69d
```

#### DNS 解析格式

```bash
# 短名稱（同 namespace）
nslookup nginx-clusterip

# 含 namespace
nslookup nginx-clusterip.exercises

# 完整 FQDN
nslookup nginx-clusterip.exercises.svc.cluster.local
```

**結果：**
```
Name:	nginx-clusterip.exercises.svc.cluster.local
Address: 10.217.4.131
```

#### Pod 的 DNS 配置

```bash
oc exec dns-tester -- cat /etc/resolv.conf
```

**輸出：**
```
search exercises.svc.cluster.local svc.cluster.local cluster.local crc.testing
nameserver 10.217.4.10
options ndots:5
```

---

## 常用指令速查表

### Service 操作

| 指令 | 說明 |
|------|------|
| `oc expose deployment/<name>` | 建立 ClusterIP Service |
| `oc get svc` | 列出 Services |
| `oc describe svc <name>` | 檢視 Service 詳情 |
| `oc get endpoints <name>` | 檢視 Service Endpoints |

### Route 操作（OpenShift）

| 指令 | 說明 |
|------|------|
| `oc expose svc/<name>` | 建立 HTTP Route |
| `oc create route edge --service=<svc>` | 建立 HTTPS Edge Route |
| `oc get routes` | 列出 Routes |
| `oc describe route <name>` | 檢視 Route 詳情 |

### NetworkPolicy 操作

| 指令 | 說明 |
|------|------|
| `oc get networkpolicy` | 列出 NetworkPolicy |
| `oc describe networkpolicy <name>` | 檢視詳情 |
| `oc delete networkpolicy <name>` | 刪除 |

### DNS 測試

```bash
# 從 Pod 內測試 DNS
oc exec <pod> -- nslookup <service-name>

# 完整格式
oc exec <pod> -- nslookup <service>.<namespace>.svc.cluster.local
```

---

## Service 類型比較

| 類型 | ClusterIP | NodePort | LoadBalancer | ExternalName |
|------|-----------|----------|--------------|--------------|
| 叢集內存取 | ✓ | ✓ | ✓ | ✓ |
| 節點 IP 存取 | ✗ | ✓ | ✓ | ✗ |
| 外部 LB | ✗ | ✗ | ✓ | ✗ |
| 使用場景 | 內部服務 | 開發測試 | 生產環境 | 外部服務別名 |

---

## Route TLS Termination 比較

| 類型 | Edge | Passthrough | Re-encrypt |
|------|------|-------------|------------|
| Router 處理 TLS | ✓ | ✗ | ✓ |
| Pod 需要 TLS | ✗ | ✓ | ✓ |
| 自動憑證 | ✓ | ✗ | 部分 |
| 使用場景 | 一般 HTTPS | 需端對端加密 | 高安全需求 |

---

*文件產生時間：2026-02-04*
*測試環境：CRC 2.57.0 / OpenShift 4.x*
