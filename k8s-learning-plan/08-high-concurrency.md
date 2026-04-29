# Kubernetes 高併發設計完整指南

> 環境：3 master + 3 worker，Ubuntu 24.04，需支援高併發流量

---

## 目錄

1. [高併發架構設計原則](#1-高併發架構設計原則)
2. [資源管理](#2-資源管理)
3. [自動擴縮容](#3-自動擴縮容)
4. [Ingress 與負載均衡](#4-ingress-與負載均衡)
5. [應用程式層面](#5-應用程式層面)
6. [效能測試](#6-效能測試)

---

## 1. 高併發架構設計原則

### 1.1 容量規劃（Capacity Planning）

在設計高併發系統之前，必須先進行容量規劃，明確以下問題：

**需求分析**
```
目標 QPS（每秒查詢數）：10,000 req/s
平均延遲目標：p50 < 50ms, p95 < 200ms, p99 < 500ms
可用性目標：99.9% SLA（每月最多 43.8 分鐘停機）
峰值流量倍數：平時的 3-5x（節日促銷等場景）
```

**節點資源評估**

```bash
# 查看目前節點資源使用狀況
kubectl top nodes
kubectl describe nodes | grep -A5 "Allocated resources"

# 計算可分配資源（扣除系統保留）
# 範例：8 core / 32GB RAM 的 worker 節點
# CPU：保留 ~10% 給 kubelet/OS，可分配約 7.2 core
# Memory：保留 ~10% + kubelet reserved，可分配約 26GB

# 計算 Pod 密度
# 假設每個 Pod request 200m CPU 和 256Mi Memory
# 每節點最多 Pod 數：min(7200m/200m, 26000Mi/256Mi) = min(36, 101) ≈ 36 個 Pod
```

**高可用性計算**

```
3 個 worker 節點，每節點最多 36 Pod
總容量：108 Pod

高可用考量（最少失去 1 節點仍可運行）：
有效容量 = 72 Pod（2/3 的容量）

如果每個服務需要 3 副本（minimum），最多可運行 24 個服務
```

### 1.2 無狀態設計原則

**原則一：Session 外部化**
```yaml
# 不好的做法：Session 存在 Pod 本地記憶體
# 好的做法：使用 Redis 存放 Session

# Redis Session 設定範例（Spring Boot）
# application.yaml
spring:
  session:
    store-type: redis
  redis:
    host: ${REDIS_HOST}
    port: 6379
    password: ${REDIS_PASS}
    
# Redis Deployment（生產環境建議使用 Redis Sentinel 或 Redis Cluster）
```

**原則二：設定外部化**
```yaml
# 使用 ConfigMap 和 Secret 而非硬編碼
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  # 應用程式設定（非機密）
  APP_ENV: "production"
  LOG_LEVEL: "info"
  WORKER_THREADS: "8"
  DB_POOL_SIZE: "20"
  DB_MAX_IDLE: "10"
  DB_CONN_TIMEOUT: "30s"
  CACHE_TTL: "300"
  RATE_LIMIT_RPS: "100"
```

**原則三：冪等設計**
```
所有 POST/PUT/DELETE 操作應設計為冪等，確保在重試、
重新排程、滾動更新時不會產生重複副作用。

實踐方式：
- 使用 idempotency key（由客戶端提供唯一 UUID）
- 資料庫操作使用 UPSERT 而非 INSERT
- 使用樂觀鎖（version/etag）防止並發衝突
```

---

## 2. 資源管理

### 2.1 Resource Requests & Limits 最佳實踐

**理解 Requests 與 Limits 的差異**

```
Request（保障值）：K8s 排程時保證分配的資源量。
                   Scheduler 根據此值決定 Pod 放在哪個節點。

Limit（上限值）：  Pod 可使用的資源上限。
                   超過 CPU limit → throttling（效能降低但不殺死）
                   超過 Memory limit → OOMKilled（立即被殺死）
```

**設定建議**

```yaml
# 高流量 API 服務的資源設定範例
resources:
  requests:
    # CPU request 設定為實際穩定負載的 50-70%
    cpu: "500m"
    # Memory request 設定為實際使用量的 80%
    memory: "512Mi"
  limits:
    # CPU limit 設定為 request 的 2-3x（允許 burst）
    # 注意：設太低會導致 throttling，影響延遲
    cpu: "1500m"
    # Memory limit 設定為 request 的 1.5-2x
    # 注意：JVM 語言需要特別注意 heap size 設定
    memory: "1Gi"
```

**Java/JVM 應用程式特別注意**

```yaml
# JVM 應用程式需要控制 Heap Size，否則容易 OOM
containers:
- name: java-app
  image: java-app:v1.0.0
  env:
  - name: JAVA_OPTS
    value: "-Xms256m -Xmx768m -XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"
  resources:
    requests:
      memory: "512Mi"
    limits:
      memory: "1Gi"
  # -Xmx768m = 75% of 1Gi limit（預留空間給 JVM overhead）
```

### 2.2 LimitRange 設定

LimitRange 為 namespace 內的容器設定預設資源值和限制範圍：

```yaml
---
# limitrange.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limitrange
  namespace: production
spec:
  limits:
  # Container 層級限制
  - type: Container
    # 預設值（未設定時自動套用）
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    # 最大值限制（防止單一 Pod 佔用過多資源）
    max:
      cpu: "4"
      memory: "8Gi"
    # 最小值限制（確保 Pod 有足夠資源）
    min:
      cpu: "50m"
      memory: "64Mi"
    # Limit/Request 最大比例（防止過度超售）
    maxLimitRequestRatio:
      cpu: "4"
      memory: "2"
  # Pod 層級限制（所有容器加總）
  - type: Pod
    max:
      cpu: "8"
      memory: "16Gi"
  # PVC 大小限制
  - type: PersistentVolumeClaim
    max:
      storage: 50Gi
    min:
      storage: 1Gi
```

### 2.3 ResourceQuota 設定

ResourceQuota 控制整個 namespace 的資源使用總量：

```yaml
---
# resourcequota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # 計算資源配額
    requests.cpu: "20"         # namespace 內所有 Pod 的 CPU request 總和
    limits.cpu: "40"
    requests.memory: "40Gi"
    limits.memory: "80Gi"
    # 儲存配額
    requests.storage: "200Gi"
    persistentvolumeclaims: "20"
    # 物件數量配額
    pods: "100"
    services: "30"
    services.loadbalancers: "3"
    services.nodeports: "0"    # 禁止使用 NodePort（安全考量）
    secrets: "50"
    configmaps: "50"
    replicationcontrollers: "0"
    # Ingress 數量
    count/ingresses.networking.k8s.io: "20"
    # Deployment 數量
    count/deployments.apps: "30"
```

### 2.4 QoS Classes 說明

```
┌─────────────────────────────────────────────────────────────────┐
│  QoS Classes（節點資源不足時，優先驅逐低優先級的 Pod）           │
├──────────────────┬──────────────────────────────────────────────┤
│  Guaranteed      │ Request == Limit（CPU 和 Memory 都相等）      │
│  最高優先級      │ 例：cpu: 500m request = 500m limit            │
│                  │ 適用：核心服務（資料庫、關鍵 API）             │
├──────────────────┼──────────────────────────────────────────────┤
│  Burstable       │ Request < Limit，或只設定其中一個             │
│  中等優先級      │ 例：cpu request: 100m, limit: 500m            │
│                  │ 適用：大多數業務服務                          │
├──────────────────┼──────────────────────────────────────────────┤
│  BestEffort      │ 完全沒有設定 Request 和 Limit                 │
│  最低優先級      │ 節點資源不足時最先被驅逐                      │
│  (避免使用)      │ 適用：批次工作、非關鍵服務                    │
└──────────────────┴──────────────────────────────────────────────┘
```

```yaml
# Guaranteed QoS 範例（核心資料庫服務）
resources:
  requests:
    cpu: "2"
    memory: "4Gi"
  limits:
    cpu: "2"        # 與 request 完全相同
    memory: "4Gi"   # 與 request 完全相同
```

---

## 3. 自動擴縮容

### 3.1 HPA（Horizontal Pod Autoscaler）

#### 3.1.1 基於 CPU/Memory 的 HPA

```yaml
---
# hpa-basic.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  # 副本數範圍
  minReplicas: 3
  maxReplicas: 20
  metrics:
  # CPU 使用率（基於 request）
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # CPU 使用率達 70% 時開始擴容
  # Memory 使用率
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  # 擴縮容行為調優
  behavior:
    # 擴容：快速擴容
    scaleUp:
      stabilizationWindowSeconds: 60  # 評估 60 秒後才擴容
      policies:
      # 每 60 秒最多增加 4 個 Pod
      - type: Pods
        value: 4
        periodSeconds: 60
      # 每 60 秒最多增加 100% 數量
      - type: Percent
        value: 100
        periodSeconds: 60
      selectPolicy: Max  # 取較大值
    # 縮容：緩慢縮容（避免流量抖動）
    scaleDown:
      stabilizationWindowSeconds: 300  # 穩定 5 分鐘後才縮容
      policies:
      - type: Pods
        value: 2
        periodSeconds: 120  # 每 2 分鐘最多縮減 2 個 Pod
      selectPolicy: Min
```

#### 3.1.2 基於自訂指標的 HPA（Prometheus Adapter）

```bash
# 安裝 Prometheus Adapter
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus-adapter prometheus-community/prometheus-adapter \
  --namespace monitoring \
  --values prometheus-adapter-values.yaml
```

**prometheus-adapter-values.yaml**

```yaml
# prometheus-adapter-values.yaml
prometheus:
  url: http://prometheus-operated.monitoring.svc
  port: 9090

rules:
  default: false
  custom:
  # 自訂指標：HTTP 請求速率
  - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
    resources:
      overrides:
        namespace: {resource: "namespace"}
        pod: {resource: "pod"}
    name:
      matches: "^(.*)_total"
      as: "${1}_per_second"
    metricsQuery: 'rate(http_requests_total{<<.LabelMatchers>>}[2m])'

  # 自訂指標：佇列長度
  - seriesQuery: 'worker_queue_length{namespace!="",pod!=""}'
    resources:
      overrides:
        namespace: {resource: "namespace"}
        pod: {resource: "pod"}
    name:
      matches: "worker_queue_length"
      as: "worker_queue_length"
    metricsQuery: 'avg(worker_queue_length{<<.LabelMatchers>>}) by (pod, namespace)'
```

```yaml
---
# 基於自訂指標（RPS）的 HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server-custom-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 3
  maxReplicas: 30
  metrics:
  # 基於每 Pod 的 HTTP 請求速率
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "500"  # 每個 Pod 每秒 500 個請求
  # 同時監控 CPU（雙重保障）
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### 3.2 VPA（Vertical Pod Autoscaler）

```bash
# 安裝 VPA
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-install.sh

# 或使用 Helm
helm repo add fairwinds-stable https://charts.fairwinds.com/stable
helm install vpa fairwinds-stable/vpa --namespace vpa --create-namespace
```

**VPA UpdateMode 說明**

```
Off      → 只提供建議，不自動修改（適合初期觀察，了解實際資源需求）
Initial  → 只在 Pod 建立時套用，不影響運行中的 Pod
Auto     → 自動驅逐並重建 Pod 以套用新的資源設定（生產環境謹慎使用）
Recreate → 類似 Auto，但更積極地重建 Pod
```

```yaml
---
# vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-server-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  updatePolicy:
    updateMode: "Off"  # 先用 Off 觀察建議值
  resourcePolicy:
    containerPolicies:
    - containerName: api-server
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 4
        memory: 8Gi
      controlledResources: ["cpu", "memory"]
```

```bash
# 查看 VPA 建議值
kubectl describe vpa api-server-vpa -n production

# 輸出範例：
# Recommendation:
#   Container Recommendations:
#     Container Name:  api-server
#     Lower Bound:
#       Cpu:     100m
#       Memory:  256Mi
#     Target:
#       Cpu:     350m      ← 建議的 request 值
#       Memory:  512Mi
#     Uncapped Target:
#       Cpu:     350m
#       Memory:  512Mi
#     Upper Bound:
#       Cpu:     2
#       Memory:  2Gi
```

### 3.3 KEDA（Kubernetes Event-Driven Autoscaling）

KEDA 允許基於各種外部事件源（Kafka、Redis、Prometheus、SQS等）進行 Pod 擴縮容。

#### 安裝 KEDA

```bash
# 使用 Helm 安裝
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

helm install keda kedacore/keda \
  --namespace keda \
  --create-namespace \
  --set resources.operator.requests.cpu=100m \
  --set resources.operator.requests.memory=128Mi \
  --set resources.operator.limits.cpu=500m \
  --set resources.operator.limits.memory=512Mi

# 確認安裝
kubectl get pods -n keda
```

#### 基於 Prometheus 指標的 ScaledObject

```yaml
---
# keda-prometheus-scaler.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: api-server-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: api-server
  # 冷卻時間（從有流量到 0 的縮容等待時間）
  cooldownPeriod: 300
  pollingInterval: 30
  minReplicaCount: 3
  maxReplicaCount: 50
  # 縮容到 0（零流量時完全縮容，適合非生產環境）
  # minReplicaCount: 0
  triggers:
  # 基於 HTTP 請求速率
  - type: prometheus
    metadata:
      serverAddress: http://prometheus-operated.monitoring:9090
      metricName: http_requests_total_per_second
      # 每個 replica 處理 500 RPS
      threshold: "500"
      query: sum(rate(http_requests_total{namespace="production",deployment="api-server"}[2m]))
  # 同時監控 P99 延遲（如延遲過高也觸發擴容）
  - type: prometheus
    metadata:
      serverAddress: http://prometheus-operated.monitoring:9090
      metricName: http_request_duration_high
      threshold: "1"  # 如果有任何 P99 > 1s 就擴容
      query: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{namespace="production"}[5m])) > 1
```

#### 基於 Kafka 的 KEDA 範例

```yaml
---
# keda-kafka-scaler.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: kafka-consumer
  minReplicaCount: 1
  maxReplicaCount: 20
  cooldownPeriod: 60
  pollingInterval: 10
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: "kafka-broker.kafka:9092"
      consumerGroup: "order-processor-group"
      topic: "order-events"
      # 每個 replica 最多處理 100 個 lag
      lagThreshold: "100"
      # 從最早的 offset 開始計算（確保 lag 計算準確）
      offsetResetPolicy: latest
```

#### 基於 Redis List 的 KEDA 範例

```yaml
---
# keda-redis-scaler.yaml
apiVersion: v1
kind: Secret
metadata:
  name: keda-redis-secret
  namespace: production
type: Opaque
stringData:
  redis-password: "your-redis-password"
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-redis-auth
  namespace: production
spec:
  secretTargetRef:
  - parameter: password
    name: keda-redis-secret
    key: redis-password
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: redis-worker-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: background-worker
  minReplicaCount: 1
  maxReplicaCount: 15
  triggers:
  - type: redis
    authenticationRef:
      name: keda-redis-auth
    metadata:
      address: "redis.production:6379"
      listName: "job-queue"
      # 每個 worker 最多處理 50 個任務
      listLength: "50"
```

### 3.4 Cluster Autoscaler（概念說明）

Cluster Autoscaler 負責在節點層級自動增減節點數量，與 HPA/KEDA 的 Pod 層級擴縮容互補。

```
工作原理：
1. 當有 Pod 因資源不足無法排程（Pending）時，CA 新增節點
2. 當節點使用率持續過低（預設 50% 以下）且 Pod 可以遷移時，CA 移除節點

與裸機環境的差異：
- 雲端環境（AWS EKS、GKE、AKS）：CA 直接呼叫雲端 API 建立/刪除 VM
- 裸機環境（本文的 3 master + 3 worker）：無法自動增減節點
  → 可以使用 Karpenter（部分雲端環境）或預先準備備用節點

裸機環境替代方案：
1. 預先準備 standby 節點（處於 shutdown 狀態），透過 IPMI/iDRAC 開機
2. 搭配 K8s node 管理腳本自動化
3. 使用 spare nodes + taint/uncordon 機制
```

---

## 4. Ingress 與負載均衡

### 4.1 Ingress Nginx Controller 安裝與調優

```bash
# 使用 Helm 安裝 Ingress Nginx
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --values ingress-nginx-values.yaml
```

**ingress-nginx-values.yaml（高併發調優）**

```yaml
# ingress-nginx-values.yaml
controller:
  # 副本數（高可用）
  replicaCount: 3
  
  # 啟用 Pod Anti-Affinity（確保分散到不同節點）
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app.kubernetes.io/name
            operator: In
            values:
            - ingress-nginx
        topologyKey: kubernetes.io/hostname

  # 資源設定
  resources:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      cpu: 2
      memory: 2Gi

  # 調優 ConfigMap（Nginx 核心參數）
  config:
    # Worker processes（通常設為 CPU 核心數，auto 會自動偵測）
    worker-processes: "auto"
    # Worker connections（每個 worker 最大連線數）
    worker-connections: "65536"
    # 啟用 upstream keepalive
    upstream-keepalive-connections: "320"
    upstream-keepalive-time: "1h"
    upstream-keepalive-timeout: "60"
    upstream-keepalive-requests: "10000"
    # 連線超時設定
    proxy-connect-timeout: "10"
    proxy-read-timeout: "120"
    proxy-send-timeout: "120"
    # 啟用 gzip 壓縮
    use-gzip: "true"
    gzip-level: "5"
    gzip-types: "application/json application/javascript text/css text/plain"
    # Body 大小限制
    proxy-body-size: "64m"
    # 緩衝區設定
    proxy-buffer-size: "16k"
    proxy-buffers-number: "8"
    # 啟用 HTTP/2
    use-http2: "true"
    # 啟用真實 IP
    use-forwarded-headers: "true"
    compute-full-forwarded-for: "true"
    # Log 格式（JSON，方便解析）
    log-format-upstream: |
      {"time":"$time_iso8601","remote_addr":"$remote_addr",
       "x_forwarded_for":"$http_x_forwarded_for",
       "request_id":"$req_id",
       "method":"$request_method","uri":"$uri",
       "status":$status,"request_time":$request_time,
       "bytes_sent":$bytes_sent,
       "upstream_addr":"$upstream_addr",
       "upstream_response_time":"$upstream_response_time"}
    # Rate Limiting 設定（nginx 層級全局限流）
    limit-req-status-code: "429"

  # Service 設定（使用 LoadBalancer 配合 MetalLB）
  service:
    type: LoadBalancer
    externalTrafficPolicy: Local  # 保留源 IP

  # HPA（Ingress Controller 本身也需要 HPA）
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 10
    targetCPUUtilizationPercentage: 70
    targetMemoryUtilizationPercentage: 80

  # Pod Disruption Budget
  podDisruptionBudget:
    enabled: true
    minAvailable: 2

  # 監控
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true
      namespace: monitoring
```

### 4.2 完整 Ingress YAML 範例（含調優 Annotations）

```yaml
---
# production-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: production-ingress
  namespace: production
  annotations:
    # Ingress class
    kubernetes.io/ingress.class: "nginx"
    # TLS 設定（使用 cert-manager 自動申請憑證）
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    # SSL 重定向
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    # 連線設定
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "120"
    # Body 大小
    nginx.ingress.kubernetes.io/proxy-body-size: "32m"
    # 啟用 gzip
    nginx.ingress.kubernetes.io/enable-gzip: "true"
    # Rate Limiting（防止單一 IP 濫用）
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "20"
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"
    # Upstream 連線池（keepalive）
    nginx.ingress.kubernetes.io/upstream-keepalive-connections: "100"
    nginx.ingress.kubernetes.io/upstream-keepalive-timeout: "60"
    # Session affinity（如需要）
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "SERVERID"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "3600"
    # CORS 設定
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.company.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
    # 安全 Headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      add_header X-Frame-Options "SAMEORIGIN" always;
      add_header X-Content-Type-Options "nosniff" always;
      add_header X-XSS-Protection "1; mode=block" always;
      add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
      add_header Referrer-Policy "strict-origin-when-cross-origin" always;
spec:
  tls:
  - hosts:
    - api.company.com
    secretName: api-company-tls
  rules:
  - host: api.company.com
    http:
      paths:
      - path: /api/v1
        pathType: Prefix
        backend:
          service:
            name: api-server
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 3000
```

### 4.3 MetalLB 安裝（裸機 LoadBalancer）

```bash
# 安裝 MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.3/config/manifests/metallb-native.yaml

# 等待 MetalLB 就緒
kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s
```

**MetalLB 設定**

```yaml
---
# metallb-config.yaml
# IP 位址池（使用環境中可用的 IP 範圍）
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: production-pool
  namespace: metallb-system
spec:
  addresses:
  # 分配給 LoadBalancer Service 的 IP 範圍
  - 192.168.100.200-192.168.100.220
  # 或使用 CIDR 格式
  # - 192.168.100.0/24
  # 自動分配（啟用後，Service 無需指定 IP）
  autoAssign: true
---
# L2 Advertisement（使用 ARP/NDP 廣播，適合大多數裸機環境）
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: production-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - production-pool
  # 限制只在特定節點廣播（可選）
  nodeSelectors:
  - matchLabels:
      node-role.kubernetes.io/worker: ""
---
# BGP Advertisement（可選，需要支援 BGP 的路由器）
# apiVersion: metallb.io/v1beta1
# kind: BGPAdvertisement
# metadata:
#   name: production-bgp
#   namespace: metallb-system
# spec:
#   ipAddressPools:
#   - production-pool
```

```yaml
# 使用指定 IP 的 Service
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  annotations:
    # 指定 MetalLB 分配特定 IP（可選）
    metallb.universe.tf/loadBalancerIPs: "192.168.100.200"
spec:
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: ingress-nginx
  ports:
  - name: http
    port: 80
    targetPort: http
  - name: https
    port: 443
    targetPort: https
  externalTrafficPolicy: Local
```

---

## 5. 應用程式層面

### 5.1 Probe 最佳實踐

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
      - name: api-server
        image: api-server:v1.0.0
        ports:
        - containerPort: 8080
        # Startup Probe：應用程式啟動時間可能較長時使用
        # 在 startupProbe 成功前，liveness/readiness probe 不會執行
        startupProbe:
          httpGet:
            path: /health/startup
            port: 8080
          failureThreshold: 30   # 最多嘗試 30 次
          periodSeconds: 10      # 每 10 秒一次 = 最多等待 5 分鐘
          successThreshold: 1
        # Readiness Probe：決定是否接收流量
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          # 初始等待時間（讓應用程式完成初始化）
          initialDelaySeconds: 10
          # 執行頻率
          periodSeconds: 10
          # 連續失敗幾次才標記為 Not Ready
          failureThreshold: 3
          # 連續成功幾次才標記為 Ready
          successThreshold: 1
          timeoutSeconds: 5
        # Liveness Probe：決定是否重啟容器
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          # 比 readiness probe 寬鬆（避免頻繁重啟）
          initialDelaySeconds: 30
          periodSeconds: 20
          failureThreshold: 5    # 允許更多次失敗才重啟
          timeoutSeconds: 10
```

**Probe Endpoint 設計建議（Go 範例）**

```go
// main.go - Health check endpoints
package main

import (
    "net/http"
    "sync/atomic"
)

var isReady atomic.Bool

// /health/startup - 只要程式能回應就成功
func startupHandler(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
}

// /health/live - 檢查程式是否活著（不要在這裡做複雜檢查）
func livenessHandler(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
}

// /health/ready - 檢查是否能處理請求（可包含依賴檢查）
func readinessHandler(w http.ResponseWriter, r *http.Request) {
    if !isReady.Load() {
        http.Error(w, "not ready", http.StatusServiceUnavailable)
        return
    }
    // 可選：快速檢查 DB 連線（設 timeout 避免阻塞）
    // 注意：不要做重的操作，否則 probe 超時
    w.WriteHeader(http.StatusOK)
}
```

### 5.2 Pod Disruption Budget（PDB）

```yaml
---
# pdb.yaml
# 確保滾動更新、節點維護時，始終保持最低副本數
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-server-pdb
  namespace: production
spec:
  # 方案 A：至少保留 2 個 Pod 可用
  minAvailable: 2
  # 方案 B：最多允許 1 個 Pod 不可用（與 A 擇一使用）
  # maxUnavailable: 1
  selector:
    matchLabels:
      app: api-server
---
# 資料庫 PDB（更嚴格）
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
  namespace: production
spec:
  # 資料庫至少要有 2/3 的節點可用（Quorum）
  minAvailable: "67%"
  selector:
    matchLabels:
      app: postgres
```

### 5.3 Topology Spread Constraints

```yaml
---
# 確保 Pod 分散到不同節點和可用區
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  replicas: 6
  template:
    spec:
      topologySpreadConstraints:
      # 跨節點均勻分散（最多允許 1 個差異）
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule  # 無法滿足時不排程（ScheduleAnyway 可放寬）
        labelSelector:
          matchLabels:
            app: api-server
      # 跨可用區分散（如有多可用區）
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway  # 可用區無法滿足時放寬
        labelSelector:
          matchLabels:
            app: api-server
```

### 5.4 Pod Anti-Affinity 設定

```yaml
---
spec:
  template:
    spec:
      affinity:
        podAntiAffinity:
          # 硬性規則：同一節點不能有兩個相同的 Pod
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - api-server
            # 以節點為單位
            topologyKey: kubernetes.io/hostname
          # 軟性規則（盡量分散，但不強制）：不同機架
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - api-server
              topologyKey: topology.kubernetes.io/zone
```

### 5.5 Rolling Update 策略調優

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      # maxSurge：滾動更新期間最多可以超出期望副本數的數量
      # 設為 2 表示最多可有 12 個 Pod（正常 10 + 額外 2 個新 Pod）
      maxSurge: 2
      # maxUnavailable：最多允許不可用的 Pod 數量
      # 設為 0 確保零停機部署（但需要額外資源）
      maxUnavailable: 0
  template:
    spec:
      containers:
      - name: api-server
        # 設定 terminationGracePeriodSeconds（Graceful Shutdown）
      terminationGracePeriodSeconds: 60
      containers:
      - name: api-server
        image: api-server:v1.0.0
        # preStop hook：在容器收到 SIGTERM 前先執行
        lifecycle:
          preStop:
            exec:
              # 等待 30 秒讓 Ingress Controller 從 endpoints 中移除此 Pod
              # 確保不再有新請求進來
              command: ["/bin/sh", "-c", "sleep 15"]
```

**Graceful Shutdown 程式設計（Go 範例）**

```go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    server := &http.Server{
        Addr:    ":8080",
        Handler: setupRoutes(),
    }

    // 背景啟動 HTTP server
    go func() {
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatalf("HTTP server error: %v", err)
        }
    }()

    // 等待終止訊號
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
    <-quit

    log.Println("收到關閉訊號，開始 Graceful Shutdown...")

    // 設定 45 秒超時（比 terminationGracePeriodSeconds 少 15 秒）
    ctx, cancel := context.WithTimeout(context.Background(), 45*time.Second)
    defer cancel()

    // 停止接受新連線，等待現有請求完成
    if err := server.Shutdown(ctx); err != nil {
        log.Printf("Graceful shutdown 失敗: %v", err)
    }
    log.Println("Server 已安全關閉")
}
```

---

## 6. 效能測試

### 6.1 k6 壓力測試腳本

```javascript
// k6-load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend, Counter } from 'k6/metrics';

// 自訂指標
const errorRate = new Rate('error_rate');
const apiDuration = new Trend('api_duration', true);
const successCounter = new Counter('success_count');

// 測試設定
export const options = {
  // 分階段壓力測試（Ramp-up → 穩定 → Spike → 縮減）
  stages: [
    // 暖機階段：5 分鐘內從 0 增加到 100 VU
    { duration: '5m', target: 100 },
    // 穩定測試：維持 100 VU 10 分鐘
    { duration: '10m', target: 100 },
    // 壓力測試：5 分鐘內增加到 500 VU
    { duration: '5m', target: 500 },
    // 峰值測試：維持 500 VU 5 分鐘
    { duration: '5m', target: 500 },
    // Spike 測試：瞬間增加到 1000 VU
    { duration: '1m', target: 1000 },
    // 恢復觀察：5 分鐘縮減
    { duration: '5m', target: 100 },
    // 降溫：縮減到 0
    { duration: '2m', target: 0 },
  ],
  // 成功標準（SLA）
  thresholds: {
    // 95% 的請求在 500ms 內完成
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    // 錯誤率低於 1%
    error_rate: ['rate<0.01'],
    // 請求成功率高於 99%
    http_req_failed: ['rate<0.01'],
  },
};

const BASE_URL = 'https://api.company.com';

export default function () {
  // 模擬真實用戶行為
  const params = {
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${__ENV.TEST_TOKEN}`,
    },
    timeout: '10s',
  };

  // 場景 1：查詢商品列表（讀取密集）
  const listRes = http.get(`${BASE_URL}/api/v1/products?page=1&limit=20`, params);
  
  const listCheck = check(listRes, {
    'products list status 200': (r) => r.status === 200,
    'products response time < 200ms': (r) => r.timings.duration < 200,
    'products has data': (r) => {
      const body = JSON.parse(r.body);
      return body.data && body.data.length > 0;
    },
  });
  
  errorRate.add(!listCheck);
  apiDuration.add(listRes.timings.duration);
  if (listCheck) successCounter.add(1);

  sleep(0.5);

  // 場景 2：建立訂單（寫入操作）
  const orderPayload = JSON.stringify({
    product_id: Math.floor(Math.random() * 1000) + 1,
    quantity: Math.floor(Math.random() * 5) + 1,
    idempotency_key: `test-${Date.now()}-${Math.random()}`,
  });

  const orderRes = http.post(`${BASE_URL}/api/v1/orders`, orderPayload, params);
  
  check(orderRes, {
    'create order status 201': (r) => r.status === 201,
    'create order response time < 500ms': (r) => r.timings.duration < 500,
  });

  sleep(1);
}

// 設定（在測試開始前執行）
export function setup() {
  console.log('開始壓力測試...');
  // 可以在這裡建立測試資料
}

// 清理（在測試結束後執行）
export function teardown(data) {
  console.log('壓力測試完成，清理測試資料...');
}
```

```bash
# 執行 k6 壓力測試
# 安裝 k6
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update && sudo apt-get install k6

# 執行測試
export TEST_TOKEN="your-test-token"
k6 run k6-load-test.js

# 輸出到 InfluxDB（配合 Grafana 展示）
k6 run --out influxdb=http://influxdb:8086/k6 k6-load-test.js

# 分散式測試（使用 k6 operator 在 K8s 內執行）
kubectl apply -f k6-test-run.yaml
```

### 6.2 Locust 設定範例

```python
# locustfile.py
from locust import HttpUser, task, between, events
import json
import random
import time

class APIUser(HttpUser):
    # 請求間隔時間（模擬真實用戶）
    wait_time = between(1, 3)
    
    def on_start(self):
        """用戶登入"""
        response = self.client.post("/api/auth/login", json={
            "email": f"testuser{random.randint(1, 1000)}@test.com",
            "password": "testpassword"
        })
        if response.status_code == 200:
            self.token = response.json()["access_token"]
            self.headers = {"Authorization": f"Bearer {self.token}"}
        else:
            self.token = None
            self.headers = {}
    
    @task(10)  # 權重 10（最常執行）
    def view_product_list(self):
        """查看商品列表"""
        with self.client.get(
            f"/api/v1/products?page={random.randint(1, 10)}&limit=20",
            headers=self.headers,
            catch_response=True,
            name="/api/v1/products"
        ) as response:
            if response.status_code == 200:
                data = response.json()
                if not data.get("data"):
                    response.failure("回應中沒有資料")
            else:
                response.failure(f"狀態碼錯誤: {response.status_code}")
    
    @task(3)   # 權重 3（中等頻率）
    def view_product_detail(self):
        """查看商品詳情"""
        product_id = random.randint(1, 500)
        self.client.get(
            f"/api/v1/products/{product_id}",
            headers=self.headers,
            name="/api/v1/products/[id]"
        )
    
    @task(1)   # 權重 1（最低頻率）
    def create_order(self):
        """建立訂單"""
        with self.client.post(
            "/api/v1/orders",
            json={
                "product_id": random.randint(1, 100),
                "quantity": random.randint(1, 5)
            },
            headers=self.headers,
            catch_response=True,
            name="/api/v1/orders"
        ) as response:
            if response.status_code not in [200, 201]:
                response.failure(f"建立訂單失敗: {response.status_code}")

# 執行指令：
# 安裝 locust
# pip install locust
# 
# 啟動 web UI
# locust -f locustfile.py --host=https://api.company.com
# 
# 無頭模式（CI/CD 用）
# locust -f locustfile.py \
#   --host=https://api.company.com \
#   --headless \
#   --users=500 \
#   --spawn-rate=50 \
#   --run-time=10m \
#   --csv=locust-results
```

### 6.3 效能指標收集與分析

```bash
# 使用 kubectl top 快速查看資源使用
kubectl top pods -n production --sort-by=cpu
kubectl top nodes

# 查詢 HPA 狀態
kubectl get hpa -n production
kubectl describe hpa api-server-hpa -n production

# 使用 Prometheus 查詢關鍵指標
# HTTP 請求速率（QPS）
rate(http_requests_total{namespace="production"}[5m])

# P95 延遲
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{namespace="production"}[5m]))

# 錯誤率
rate(http_requests_total{namespace="production",status=~"5.."}[5m]) 
  / rate(http_requests_total{namespace="production"}[5m])

# CPU 節流比例（值越高代表 CPU limit 設太低）
rate(container_cpu_cfs_throttled_seconds_total{namespace="production"}[5m])
  / rate(container_cpu_cfs_periods_total{namespace="production"}[5m])

# 記憶體使用率
container_memory_working_set_bytes{namespace="production"}
  / container_spec_memory_limit_bytes{namespace="production"}
```

**效能問題排查清單**

```
高 CPU 節流（throttling > 20%）
→ 增加 CPU limit 或調整 HPA 目標，讓擴容更積極

高記憶體使用率（> 85%）
→ 排查記憶體洩漏，或調整 memory limit/request

高延遲（P95 > 目標值）
→ 檢查 upstream 服務、資料庫查詢、連線池設定

Pod 頻繁重啟（OOMKilled）
→ 增加 memory limit，排查記憶體洩漏

HPA 無法擴容（stuck at maxReplicas）
→ 增加 maxReplicas，或增加節點資源
```
