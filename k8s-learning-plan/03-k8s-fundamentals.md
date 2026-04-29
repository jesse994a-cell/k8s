# Kubernetes 核心概念與基礎知識

> 環境：Ubuntu 24.04 | K8s 3 master + 3 worker | CNI: Calico | CRI: containerd

---

## 目錄

1. [Kubernetes 架構概覽](#1-kubernetes-架構概覽)
2. [Control Plane 元件詳解](#2-control-plane-元件詳解)
3. [Worker Node 元件詳解](#3-worker-node-元件詳解)
4. [核心資源物件說明](#4-核心資源物件說明)
5. [kubectl 常用指令大全](#5-kubectl-常用指令大全)
6. [YAML 撰寫規範與最佳實踐](#6-yaml-撰寫規範與最佳實踐)
7. [資源請求與限制](#7-資源請求與限制)
8. [標籤與選擇器](#8-標籤與選擇器)
9. [實作練習：Nginx + Service + Ingress](#9-實作練習nginx--service--ingress)

---

## 1. Kubernetes 架構概覽

Kubernetes（簡稱 K8s）是一個開源的容器編排平台，負責自動化容器應用程式的部署、擴展與管理。

### 1.1 整體架構圖（本環境 HA 配置）

```
                         ┌─────────────────────────────────────┐
                         │        External / Users / CI        │
                         └──────────────┬──────────────────────┘
                                        │ kubectl / API
                                        ▼
                         ┌─────────────────────────────────────┐
                         │   VIP: 192.168.100.10:6443          │
                         │   HAProxy + Keepalived (on masters) │
                         └──────┬───────────┬───────────┬──────┘
                                │           │           │
               ┌────────────────▼──┐  ┌─────▼──────┐  ┌▼───────────────┐
               │  master1          │  │  master2   │  │  master3       │
               │  192.168.100.11   │  │  .100.12   │  │  .100.13       │
               │ ┌───────────────┐ │  │ ┌────────┐ │  │ ┌───────────┐  │
               │ │ kube-apiserver│ │  │ │kube-api│ │  │ │kube-api   │  │
               │ │ kube-scheduler│ │  │ │schedul │ │  │ │scheduler  │  │
               │ │ kube-ctrl-mgr │ │  │ │ctrl-mgr│ │  │ │ctrl-mgr   │  │
               │ │ etcd          │ │  │ │etcd    │ │  │ │etcd       │  │
               │ └───────────────┘ │  │ └────────┘ │  │ └───────────┘  │
               └───────────────────┘  └────────────┘  └────────────────┘
                                        │           │
                    ┌───────────────────┼───────────┼──────────────────┐
                    │                   │           │                  │
         ┌──────────▼────────┐  ┌───────▼────────┐  ┌────────────────▼─┐
         │  worker1          │  │  worker2       │  │  worker3         │
         │  192.168.100.21   │  │  .100.22       │  │  .100.23         │
         │ ┌───────────────┐ │  │ ┌────────────┐ │  │ ┌─────────────┐  │
         │ │ kubelet       │ │  │ │ kubelet    │ │  │ │ kubelet     │  │
         │ │ kube-proxy    │ │  │ │ kube-proxy │ │  │ │ kube-proxy  │  │
         │ │ containerd    │ │  │ │ containerd │ │  │ │ containerd  │  │
         │ │ [Pods...]     │ │  │ │ [Pods...]  │ │  │ │ [Pods...]   │  │
         │ └───────────────┘ │  │ └────────────┘ │  │ └─────────────┘  │
         └───────────────────┘  └────────────────┘  └──────────────────┘
```

### 1.2 K8s 的核心設計理念

| 概念 | 說明 |
|------|------|
| 宣告式 API | 使用者描述「期望狀態」，K8s 負責達成 |
| 自我修復 | Pod 崩潰時自動重啟，節點故障時重新排程 |
| 水平擴展 | 透過 ReplicaSet / HPA 動態調整副本數 |
| 服務發現 | kube-dns + Service 提供內部 DNS 解析 |
| 滾動更新 | Deployment 支援零停機升級與回滾 |

---

## 2. Control Plane 元件詳解

Control Plane（控制平面）是 K8s 的大腦，負責管理整個叢集的狀態。在本環境中，3 台 master 節點各自運行完整的 Control Plane 元件。

### 2.1 kube-apiserver

**角色**：K8s 叢集的唯一入口，所有操作都透過 RESTful API 進行。

**主要功能**：
- 驗證（Authentication）與授權（Authorization）所有 API 請求
- 資料讀寫操作，最終持久化至 etcd
- 作為其他元件之間的通訊中介
- 支援 Watch 機制，允許元件監聽資源變化

**重要特性**：
- 本環境透過 HAProxy + Keepalived VIP（192.168.100.10:6443）提供 HA
- 預設監聽 6443 port（HTTPS）
- 無狀態服務，可水平擴展

```bash
# 查看 kube-apiserver 狀態
kubectl get pod kube-apiserver-master1 -n kube-system

# 查看 apiserver 日誌
kubectl logs kube-apiserver-master1 -n kube-system --tail=50
```

### 2.2 etcd

**角色**：分散式鍵值儲存，是 K8s 所有叢集狀態資料的唯一真實來源（Single Source of Truth）。

**主要功能**：
- 儲存所有 K8s 物件（Pod、Service、Deployment 等）
- 使用 Raft 共識演算法保證資料一致性
- 支援 Watch 機制，資料變更時通知 kube-apiserver

**本環境 etcd 叢集**：
- 採用 stacked etcd（etcd 與 control plane 共同部署在 master 節點）
- 3 個 etcd 節點，quorum = 2（可容忍 1 個節點故障）

```bash
# 查看 etcd 叢集成員（在 master 節點執行）
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list

# 查看 etcd 健康狀態
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://192.168.100.11:2379,https://192.168.100.12:2379,https://192.168.100.13:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### 2.3 kube-scheduler

**角色**：負責監聽未被排程的 Pod，並為其選擇最合適的 Node。

**排程流程**：
1. **過濾（Filtering）**：排除不符合 Pod 需求的節點（資源不足、taints 不符等）
2. **評分（Scoring）**：對剩餘節點打分，選出最佳節點
3. **綁定（Binding）**：將 Pod 與選定節點綁定

**影響排程的因素**：
- 資源請求（CPU/Memory requests）
- Node Selector / Node Affinity
- Pod Affinity / Anti-affinity
- Taints and Tolerations
- Pod 優先級（PriorityClass）

```bash
# 查看 scheduler 日誌
kubectl logs kube-scheduler-master1 -n kube-system --tail=30

# 查看 Pod 排程事件
kubectl describe pod <pod-name> | grep -A 10 Events
```

### 2.4 kube-controller-manager

**角色**：運行多個控制器（Controller），持續監控叢集狀態，確保實際狀態符合期望狀態。

**主要控制器**：

| 控制器 | 功能說明 |
|--------|----------|
| Node Controller | 監控節點狀態，處理節點離線事件 |
| Replication Controller | 確保 ReplicaSet 維持指定副本數 |
| Endpoints Controller | 維護 Service 與 Pod 的端點對應 |
| ServiceAccount Controller | 為新 Namespace 建立預設 ServiceAccount |
| Deployment Controller | 管理 Deployment 的滾動更新 |
| StatefulSet Controller | 管理有狀態應用的部署與擴展 |
| DaemonSet Controller | 確保指定節點都運行 DaemonSet Pod |
| Job/CronJob Controller | 管理批次任務的執行 |

```bash
# 查看 controller-manager 日誌
kubectl logs kube-controller-manager-master1 -n kube-system --tail=30
```

---

## 3. Worker Node 元件詳解

Worker Node 是實際運行應用程式容器的節點，本環境有 3 台 worker（192.168.100.21~23）。

### 3.1 kubelet

**角色**：運行在每個節點上的 Node Agent，負責管理該節點上的 Pod 生命週期。

**主要功能**：
- 向 kube-apiserver 註冊節點
- 監聽 PodSpec，確保容器按規格運行
- 透過 CRI（Container Runtime Interface）與 containerd 互動
- 定期向 kube-apiserver 回報節點狀態與資源使用量
- 執行 liveness probe、readiness probe、startup probe

```bash
# 查看 kubelet 服務狀態
systemctl status kubelet

# 查看 kubelet 日誌
journalctl -u kubelet -f --since "10 minutes ago"

# 查看節點資源使用（需 metrics-server）
kubectl top node
kubectl describe node worker1
```

### 3.2 kube-proxy

**角色**：運行在每個節點上，負責實現 K8s Service 的網路規則。

**主要功能**：
- 監聽 Service 和 Endpoints 的變化
- 維護 iptables 或 IPVS 規則，實現 Pod 間的負載均衡
- 處理 ClusterIP、NodePort、LoadBalancer 的流量轉發

**運作模式**：
- **iptables 模式**（預設）：透過 iptables 規則轉發流量，隨機選擇後端
- **IPVS 模式**：效能更好，支援更多負載均衡演算法，適合大型叢集

```bash
# 查看 kube-proxy 設定
kubectl get configmap kube-proxy -n kube-system -o yaml

# 查看 iptables 規則（在節點上執行）
iptables -t nat -L KUBE-SERVICES -n | head -20
```

### 3.3 Container Runtime（containerd）

**角色**：實際負責容器的建立、啟動、停止、刪除操作。

**本環境使用 containerd**，透過 CRI 介面與 kubelet 溝通。

**containerd 架構**：
```
kubelet -> CRI gRPC -> containerd -> runc -> Linux Namespaces/Cgroups
```

```bash
# 查看 containerd 服務狀態
systemctl status containerd

# 使用 crictl 查看容器（類似 docker 指令）
crictl ps
crictl images
crictl pods

# 查看 containerd 設定
cat /etc/containerd/config.toml
```

---

## 4. 核心資源物件說明

### 4.1 Pod

Pod 是 K8s 中最小的可部署單元，一個 Pod 可以包含一或多個容器，這些容器共享網路命名空間和儲存 Volume。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: default
  labels:
    app: nginx
    version: "1.25"
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "200m"
        memory: "256Mi"
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 5
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 3
```

**Pod 生命週期狀態**：

| 狀態 | 說明 |
|------|------|
| Pending | Pod 已建立但容器尚未啟動（等待排程或拉取映像檔） |
| Running | Pod 已排程到節點，至少一個容器在運行 |
| Succeeded | 所有容器成功終止（適用 Job） |
| Failed | 所有容器終止，且至少一個以失敗狀態終止 |
| Unknown | 無法取得 Pod 狀態（通常是節點網路問題） |

### 4.2 ReplicaSet

確保指定數量的 Pod 副本持續運行。通常不直接使用，而是透過 Deployment 管理。

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

### 4.3 Deployment

管理 Pod 的宣告式更新，支援滾動更新、回滾等功能。是部署無狀態應用的標準方式。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 升級時最多多幾個 Pod
      maxUnavailable: 0  # 升級時最多幾個 Pod 不可用
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

```bash
# 更新映像版本（觸發滾動更新）
kubectl set image deployment/nginx-deployment nginx=nginx:1.26

# 查看更新進度
kubectl rollout status deployment/nginx-deployment

# 查看更新歷史
kubectl rollout history deployment/nginx-deployment

# 回滾到上一版
kubectl rollout undo deployment/nginx-deployment

# 回滾到指定版本
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

### 4.4 StatefulSet

用於部署有狀態應用（如資料庫），提供穩定的網路識別碼和持久化儲存。

**與 Deployment 的差異**：

| 特性 | Deployment | StatefulSet |
|------|-----------|-------------|
| Pod 名稱 | 隨機後綴（如 nginx-abc12） | 有序後綴（如 mysql-0, mysql-1） |
| 啟動順序 | 並行啟動 | 依序啟動（0, 1, 2...） |
| 儲存 | 共用 Volume | 每個 Pod 獨立的 PVC |
| DNS | 無固定 DNS | 每個 Pod 有固定 DNS |

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"
  replicas: 3
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
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
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
          storage: 10Gi
```

### 4.5 DaemonSet

確保所有（或指定的）節點上都運行一個 Pod 副本。常用於日誌收集、監控 Agent、網路插件等。

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:latest
        ports:
        - containerPort: 9100
          hostPort: 9100
      tolerations:
      - operator: Exists   # 允許在所有節點（包含 master）上運行
```

### 4.6 Service

Service 為一組 Pod 提供穩定的網路存取入口，解決 Pod IP 動態變化的問題。

#### ClusterIP（預設）

叢集內部存取，外部無法直接存取。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80        # Service 監聽的 Port
    targetPort: 80  # 轉發到 Pod 的 Port
```

#### NodePort

在每個節點上開放指定 Port（30000-32767），外部可透過 `<NodeIP>:<NodePort>` 存取。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080   # 不指定則隨機分配
```

#### LoadBalancer

搭配雲端供應商或 MetalLB，自動建立外部負載均衡器。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

### 4.7 ConfigMap

儲存非機密的設定資料（環境變數、設定檔等），與容器映像解耦。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  nginx.conf: |
    server {
      listen 80;
      server_name example.com;
      location / {
        root /usr/share/nginx/html;
      }
    }
```

```yaml
# 在 Pod 中使用 ConfigMap（環境變數 + 掛載檔案）
spec:
  containers:
  - name: app
    image: myapp:1.0
    envFrom:
    - configMapRef:
        name: app-config
    volumeMounts:
    - name: config-vol
      mountPath: /etc/nginx/conf.d
  volumes:
  - name: config-vol
    configMap:
      name: app-config
      items:
      - key: nginx.conf
        path: default.conf
```

### 4.8 Secret

儲存敏感資料（密碼、Token、憑證等），以 Base64 編碼儲存（注意：不是加密，應搭配 RBAC 和 etcd 加密使用）。

```bash
# 建立 Secret（指令方式，自動 Base64 編碼）
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=MyP@ssw0rd

# 建立 TLS Secret
kubectl create secret tls my-tls-secret \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=           # echo -n "admin" | base64
  password: TXlQQHNzdzByZA==   # echo -n "MyP@ssw0rd" | base64
```

### 4.9 PersistentVolume 與 PersistentVolumeClaim

**PersistentVolume（PV）**：管理員預先建立或動態佈建的儲存資源。
**PersistentVolumeClaim（PVC）**：使用者申請儲存資源的請求。

```yaml
# PersistentVolume 範例（NFS）
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 192.168.100.50
    path: /exports/data
---
# PersistentVolumeClaim 範例
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
```

**Access Modes 說明**：

| 模式 | 縮寫 | 說明 |
|------|------|------|
| ReadWriteOnce | RWO | 單一節點讀寫 |
| ReadOnlyMany | ROX | 多節點唯讀 |
| ReadWriteMany | RWX | 多節點讀寫 |
| ReadWriteOncePod | RWOP | 單一 Pod 讀寫（K8s 1.22+）|

### 4.10 Namespace

提供資源的邏輯隔離，適合多團隊或多環境的場景。

```bash
# 建立 Namespace
kubectl create namespace development
kubectl create namespace production

# 查看所有 Namespace
kubectl get namespaces

# 在特定 Namespace 操作
kubectl get pods -n development
kubectl get all -n production

# 設定預設 Namespace（避免每次加 -n）
kubectl config set-context --current --namespace=development
```

### 4.11 RBAC（Role-Based Access Control）

K8s 的授權機制，透過角色與綁定控制誰能對哪些資源執行什麼操作。

**四種核心物件**：

| 物件 | 作用範圍 | 說明 |
|------|----------|------|
| Role | Namespace | 定義 Namespace 內的權限 |
| ClusterRole | 全叢集 | 定義全叢集範圍的權限 |
| RoleBinding | Namespace | 將 Role 綁定到使用者/ServiceAccount |
| ClusterRoleBinding | 全叢集 | 將 ClusterRole 綁定到使用者/ServiceAccount |

```yaml
# Role：允許讀取 development namespace 的 Pod
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: development
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
---
# RoleBinding：將 Role 綁定給 dev-user
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: development
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
# 驗證權限
kubectl auth can-i get pods --namespace=development --as=dev-user
kubectl auth can-i delete deployments --namespace=development --as=dev-user
```

### 4.12 Ingress 與 IngressController

**Ingress**：定義 HTTP/HTTPS 流量的路由規則（路徑、域名路由、TLS 終止）。
**IngressController**：實際執行 Ingress 規則的元件（如 Nginx Ingress Controller）。

```
Internet -> IngressController (NodePort/LB) -> Ingress 規則解析 -> Service -> Pods
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls-secret
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### 4.13 HorizontalPodAutoscaler（HPA）

根據 CPU、Memory 或自訂指標，自動調整 Deployment/StatefulSet 的副本數。

**前置需求**：Metrics Server 必須已安裝。

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70    # CPU 使用率超過 70% 時擴展
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

```bash
# 查看 HPA 狀態
kubectl get hpa
kubectl describe hpa nginx-hpa
```

---

## 5. kubectl 常用指令大全

### 5.1 叢集資訊

```bash
# 查看叢集資訊
kubectl cluster-info

# 查看所有節點
kubectl get nodes
kubectl get nodes -o wide          # 顯示 IP、OS 等詳細資訊

# 查看節點資源使用（需 metrics-server）
kubectl top nodes

# 查看 kubeconfig 設定
kubectl config view
kubectl config get-contexts         # 列出所有 context
kubectl config use-context <name>   # 切換 context
```

### 5.2 Pod 操作

```bash
# 列出 Pod
kubectl get pods                             # 預設 namespace
kubectl get pods -n kube-system              # 指定 namespace
kubectl get pods --all-namespaces            # 所有 namespace（可用 -A）
kubectl get pods -o wide                     # 顯示 Node IP 等詳情
kubectl get pods -l app=nginx                # 用 Label 過濾
kubectl get pods -w                          # Watch 模式（即時更新）

# 查看 Pod 詳情
kubectl describe pod <pod-name>

# 查看 Pod 日誌
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>  # 多容器 Pod 指定容器
kubectl logs <pod-name> -f                   # Follow 模式
kubectl logs <pod-name> --previous           # 查看崩潰前的容器日誌
kubectl logs <pod-name> --tail=100           # 只顯示最後 100 行

# 進入 Pod 執行指令
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec -it <pod-name> -c <container> -- /bin/sh

# 複製檔案
kubectl cp <pod-name>:/path/to/file ./local-file   # Pod -> 本機
kubectl cp ./local-file <pod-name>:/path/to/file   # 本機 -> Pod

# Port Forward（本機端口轉發到 Pod）
kubectl port-forward pod/<pod-name> 8080:80
kubectl port-forward service/<svc-name> 8080:80
```

### 5.3 Deployment 操作

```bash
# 建立/更新資源
kubectl apply -f deployment.yaml
kubectl apply -f ./manifests/             # 套用目錄下所有 yaml

# 快速建立 Deployment
kubectl create deployment nginx --image=nginx:1.25 --replicas=3

# 擴縮容
kubectl scale deployment nginx-deployment --replicas=5

# 更新映像
kubectl set image deployment/nginx-deployment nginx=nginx:1.26

# 查看滾動更新狀態
kubectl rollout status deployment/nginx-deployment

# 暫停/恢復滾動更新
kubectl rollout pause deployment/nginx-deployment
kubectl rollout resume deployment/nginx-deployment

# 查看更新歷史
kubectl rollout history deployment/nginx-deployment

# 回滾操作
kubectl rollout undo deployment/nginx-deployment
kubectl rollout undo deployment/nginx-deployment --to-revision=3

# 刪除資源
kubectl delete deployment nginx-deployment
kubectl delete -f deployment.yaml
kubectl delete pod <pod-name> --force --grace-period=0   # 強制刪除
```

### 5.4 資源查看與編輯

```bash
# 取得資源定義
kubectl get deployment nginx-deployment -o yaml
kubectl get deployment nginx-deployment -o json
kubectl get pod <pod-name> -o jsonpath='{.spec.nodeName}'

# 直接編輯資源（開啟編輯器）
kubectl edit deployment nginx-deployment

# 查看所有支援的資源類型
kubectl api-resources

# 查看資源欄位說明（非常實用）
kubectl explain pod
kubectl explain pod.spec.containers
kubectl explain deployment.spec.strategy

# 查看事件（排障很有用）
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get events -n kube-system
```

### 5.5 除錯指令

```bash
# 在節點上執行除錯 Pod
kubectl debug node/worker1 -it --image=ubuntu

# 建立除錯用的臨時容器（K8s 1.23+）
kubectl debug -it <pod-name> --image=busybox --target=<container-name>

# 查看 Pod 資源使用
kubectl top pod
kubectl top pod <pod-name> --containers

# 查看 Pod 所在節點
kubectl get pod <pod-name> -o wide

# 查看 Service Endpoints
kubectl get endpoints <service-name>
kubectl describe endpoints <service-name>

# 強制重建 Pod（Deployment 會自動重建）
kubectl delete pod <pod-name>

# 暫停節點排程（維護用）
kubectl cordon worker1
kubectl drain worker1 --ignore-daemonsets --delete-emptydir-data
kubectl uncordon worker1    # 恢復排程
```

---

## 6. YAML 撰寫規範與最佳實踐

### 6.1 必填欄位

每個 K8s 資源 YAML 都必須包含以下 4 個頂層欄位：

```yaml
apiVersion: apps/v1        # API 群組/版本（kubectl api-resources 可查）
kind: Deployment           # 資源類型
metadata:                  # 資源元資料
  name: my-app
  namespace: default
spec:                      # 資源規格（期望狀態）
  ...
```

### 6.2 推薦的完整 Deployment 範本

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
    version: "1.0.0"
    team: backend
  annotations:
    kubernetes.io/change-cause: "Initial deployment"
    description: "My application deployment"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app           # 必須與 template.metadata.labels 一致
  template:
    metadata:
      labels:
        app: my-app
        version: "1.0.0"
    spec:
      # 安全性設定
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000

      containers:
      - name: my-app
        image: my-app:1.0.0
        imagePullPolicy: IfNotPresent   # Always / Never / IfNotPresent

        ports:
        - name: http
          containerPort: 8080
          protocol: TCP

        # 環境變數
        env:
        - name: APP_ENV
          value: "production"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password

        # 資源限制（強烈建議設定）
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"

        # 健康檢查
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5

      # 反親和性（分散 Pod 到不同節點）
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: my-app
              topologyKey: kubernetes.io/hostname

      terminationGracePeriodSeconds: 30
```

### 6.3 最佳實踐清單

| 項目 | 建議 |
|------|------|
| 映像版本 | 不要使用 `latest`，指定明確版本（如 `nginx:1.25.3`）|
| 資源限制 | 每個容器都設定 requests 和 limits |
| 健康檢查 | 設定 livenessProbe 和 readinessProbe |
| 安全性 | 設定 securityContext，避免以 root 執行 |
| 標籤 | 為所有資源加上有意義的 Labels |
| Namespace | 不同環境/應用使用不同 Namespace |
| Secret | 敏感資料使用 Secret，不要硬編碼在 YAML |
| 副本數 | 生產環境 Deployment replicas >= 2 |
| 反親和性 | 使用 podAntiAffinity 分散 Pod 到不同節點 |
| 滾動更新 | 設定 maxSurge 和 maxUnavailable 確保零停機 |

---

## 7. 資源請求與限制

### 7.1 概念說明

```
requests（請求）：排程時使用，kubelet 確保節點有足夠資源才排程 Pod
limits（限制）：運行時使用，超出 CPU limit 會被節流，超出 Memory limit 會被 OOMKill
```

### 7.2 CPU 單位

| 表示方式 | 等效說明 |
|----------|----------|
| `1` | 1 個完整 CPU 核心 |
| `1000m` | 1 個完整 CPU 核心（毫核） |
| `500m` | 0.5 個 CPU 核心 |
| `100m` | 0.1 個 CPU 核心 |

### 7.3 Memory 單位

| 單位 | 大小 |
|------|------|
| `Ki` | Kibibyte = 1024 bytes |
| `Mi` | Mebibyte = 1024 Ki |
| `Gi` | Gibibyte = 1024 Mi |

### 7.4 QoS Class（服務品質等級）

K8s 根據 requests/limits 設定自動分配 QoS 等級，影響節點資源不足時的驅逐順序：

| QoS Class | 條件 | 驅逐優先順序 |
|-----------|------|------------|
| Guaranteed | requests == limits（CPU + Memory 都相等）| 最後被驅逐 |
| Burstable | 有設定 requests 或 limits（但不相等）| 中間 |
| BestEffort | 未設定任何 requests/limits | 最先被驅逐 |

### 7.5 LimitRange（Namespace 層級預設值）

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: development
spec:
  limits:
  - type: Container
    default:             # 未設定 limits 時的預設值
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:      # 未設定 requests 時的預設值
      cpu: "100m"
      memory: "128Mi"
    max:                 # 允許的最大值
      cpu: "2"
      memory: "2Gi"
    min:                 # 允許的最小值
      cpu: "50m"
      memory: "64Mi"
```

### 7.6 ResourceQuota（Namespace 資源配額）

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: development-quota
  namespace: development
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    count/pods: "50"
    count/services: "20"
    count/deployments.apps: "20"
    count/persistentvolumeclaims: "10"
```

---

## 8. 標籤與選擇器

### 8.1 Labels（標籤）

Labels 是附加在 K8s 物件上的鍵值對，用於組織和選擇資源。

**官方建議的標籤規範（app.kubernetes.io 前綴）**：

```yaml
metadata:
  labels:
    app.kubernetes.io/name: "my-app"           # 應用名稱
    app.kubernetes.io/version: "1.0.0"         # 版本
    app.kubernetes.io/component: "backend"     # 元件角色
    app.kubernetes.io/part-of: "my-system"     # 所屬系統
    app.kubernetes.io/managed-by: "helm"       # 管理工具
    environment: "production"                  # 環境
    team: "platform"                           # 負責團隊
```

### 8.2 Selectors（選擇器）

選擇器用於根據 Labels 篩選資源。

```bash
# 等式選擇（Equality-based）
kubectl get pods -l app=nginx
kubectl get pods -l app=nginx,env=production   # AND 條件

# 集合選擇（Set-based）
kubectl get pods -l 'app in (nginx, apache)'
kubectl get pods -l 'env notin (development, staging)'
kubectl get pods -l '!deprecated'             # 不含此 label 的資源
```

```yaml
# 在資源定義中使用 matchExpressions
selector:
  matchLabels:
    app: nginx
  matchExpressions:
  - key: environment
    operator: In
    values: [production, staging]
  - key: deprecated
    operator: DoesNotExist
```

### 8.3 Annotations（注釋）

Annotations 也是鍵值對，但不用於選擇，用於儲存非識別性的輔助資訊。

```yaml
metadata:
  annotations:
    kubernetes.io/change-cause: "Update nginx to 1.26"
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    description: "Frontend web server"
    last-modified-by: "ops-team"
    git-commit: "abc123def456"
```

### 8.4 Node Selector 與 Node Affinity

```yaml
# 簡單的 nodeSelector（等式匹配）
spec:
  nodeSelector:
    kubernetes.io/arch: amd64
    node-role: worker

# 進階的 nodeAffinity（支援集合匹配與軟性要求）
spec:
  affinity:
    nodeAffinity:
      # 強制要求（硬性）
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-role
            operator: In
            values: [worker, compute]
      # 偏好要求（軟性）
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values: [zone-a]
```

### 8.5 Taints 與 Tolerations

```bash
# 為節點加上 Taint（禁止一般 Pod 排程）
kubectl taint nodes master1 node-role.kubernetes.io/control-plane:NoSchedule

# 查看節點 Taint
kubectl describe node master1 | grep Taint

# 移除 Taint
kubectl taint nodes master1 node-role.kubernetes.io/control-plane:NoSchedule-
```

```yaml
# Pod 設定 Toleration（允許排程到有 Taint 的節點）
spec:
  tolerations:
  - key: "node-role.kubernetes.io/control-plane"
    operator: "Exists"
    effect: "NoSchedule"
```

---

## 9. 實作練習：Nginx + Service + Ingress

本節透過實際操作，部署一個完整的 Nginx 應用，包含 Deployment、Service 和 Ingress。

### 步驟 1：建立 Namespace

```bash
kubectl create namespace demo
kubectl config set-context --current --namespace=demo
```

### 步驟 2：建立 ConfigMap（自訂 Nginx 首頁）

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-html
  namespace: demo
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>K8s Demo</title></head>
    <body>
      <h1>Hello from Kubernetes!</h1>
      <p>Served by: nginx-demo</p>
    </body>
    </html>
EOF
```

### 步驟 3：建立 Deployment

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  namespace: demo
  labels:
    app: nginx-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-demo
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "100m"
            memory: "128Mi"
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 3
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
      volumes:
      - name: html
        configMap:
          name: nginx-html
EOF
```

### 步驟 4：建立 Service

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx-demo-svc
  namespace: demo
spec:
  type: ClusterIP
  selector:
    app: nginx-demo
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
EOF
```

### 步驟 5：安裝 Nginx Ingress Controller

```bash
# 安裝 Nginx Ingress Controller（Bare Metal 版本）
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml

# 等待 ingress-nginx 就緒
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s

# 查看 NodePort
kubectl get svc -n ingress-nginx
```

### 步驟 6：建立 Ingress

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-demo-ingress
  namespace: demo
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: demo.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-demo-svc
            port:
              number: 80
EOF
```

### 步驟 7：驗證部署

```bash
# 確認所有資源狀態
kubectl get all -n demo

# 確認 Pod 全部 Running
kubectl get pods -n demo
# 預期：3 個 nginx-demo-xxxxx Pod 均為 Running 1/1

# 確認 Service Endpoints 有值
kubectl get endpoints nginx-demo-svc -n demo

# 確認 Ingress
kubectl get ingress -n demo
kubectl describe ingress nginx-demo-ingress -n demo

# 本機測試（取得 ingress NodePort）
NODEPORT=$(kubectl get svc ingress-nginx-controller -n ingress-nginx \
  -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')
echo "Ingress NodePort: $NODEPORT"

# 透過任意 worker 節點 IP 測試（需加 Host header）
curl -H "Host: demo.local" http://192.168.100.21:$NODEPORT/
```

### 步驟 8：測試自動修復

```bash
# 刪除一個 Pod，觀察 Deployment 自動重建
kubectl delete pod -l app=nginx-demo -n demo --wait=false

# 即時監看 Pod 狀態（Ctrl+C 離開）
kubectl get pods -n demo -w
```

### 步驟 9：測試滾動更新

```bash
# 更新到新版本
kubectl set image deployment/nginx-demo nginx=nginx:1.26 -n demo

# 觀察滾動更新過程
kubectl rollout status deployment/nginx-demo -n demo

# 確認版本
kubectl get deployment nginx-demo -n demo -o jsonpath='{.spec.template.spec.containers[0].image}'

# 若有問題，回滾
kubectl rollout undo deployment/nginx-demo -n demo
```

### 步驟 10：設定 HPA（自動擴縮）

```bash
# 確認 metrics-server 已安裝
kubectl top node

# 建立 HPA
kubectl autoscale deployment nginx-demo \
  --min=2 --max=8 --cpu-percent=50 \
  -n demo

# 查看 HPA 狀態
kubectl get hpa -n demo
kubectl describe hpa nginx-demo -n demo
```

### 步驟 11：清理

```bash
# 刪除所有 demo 資源
kubectl delete namespace demo

# 還原預設 namespace
kubectl config set-context --current --namespace=default
```

---

## 參考資源

- [Kubernetes 官方文件](https://kubernetes.io/docs/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [K8s API Reference](https://kubernetes.io/docs/reference/kubernetes-api/)
- [Calico 文件](https://docs.tigera.io/calico/latest/about/)
- [containerd 文件](https://containerd.io/docs/)
- [Nginx Ingress Controller](https://kubernetes.github.io/ingress-nginx/)

---

*文件版本：1.0 | 更新日期：2026-04-29 | 適用 K8s 版本：1.28+*
