# Kubernetes 監控與日誌管理完整指南

> 環境：3 master + 3 worker，Ubuntu 24.04，GitLab + ArgoCD CI/CD

---

## 目錄

1. [監控架構](#1-監控架構)
2. [日誌管理（Loki Stack）](#2-日誌管理loki-stack)
3. [分散式追蹤](#3-分散式追蹤)
4. [應用程式監控最佳實踐](#4-應用程式監控最佳實踐)

---

## 1. 監控架構

### 1.1 整體架構說明

```
┌──────────────────────────────────────────────────────────────────┐
│  資料收集層                                                        │
│  Node Exporter (每節點) → Prometheus (時序資料庫)                  │
│  cAdvisor (容器指標)    ↗                                          │
│  kube-state-metrics    ↗ （K8s 物件狀態）                         │
│  應用程式 /metrics      ↗ （業務指標）                             │
└──────────────────────────────────┬───────────────────────────────┘
                                   │ 查詢 / 告警
┌──────────────────────────────────▼───────────────────────────────┐
│  展示與告警層                                                       │
│  Grafana （Dashboard 可視化）                                       │
│  Alertmanager （告警路由：Slack / PagerDuty / Email）               │
└──────────────────────────────────────────────────────────────────┘
```

**各元件功能說明**

| 元件                | 功能                                                |
|---------------------|-----------------------------------------------------|
| Prometheus          | 時序資料庫，定期拉取（scrape）各端點的 metrics      |
| Node Exporter       | 提供節點硬體和 OS 指標（CPU、Memory、Disk、Network）|
| kube-state-metrics  | 提供 K8s 物件狀態指標（Pod 數量、Deployment 狀態等）|
| cAdvisor            | 提供容器資源使用指標（已整合進 kubelet）            |
| Grafana             | 儀表板展示，支援多種資料源                          |
| Alertmanager        | 告警去重、分組、路由到不同通道                      |

### 1.2 kube-prometheus-stack 安裝

```bash
# 加入 Prometheus Community Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# 建立 monitoring namespace
kubectl create namespace monitoring

# 安裝 kube-prometheus-stack（包含 Prometheus、Grafana、Alertmanager、Node Exporter、kube-state-metrics）
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values kube-prometheus-stack-values.yaml \
  --wait
```

**kube-prometheus-stack-values.yaml（完整設定）**

```yaml
# kube-prometheus-stack-values.yaml

# Grafana 設定
grafana:
  enabled: true
  adminUser: admin
  adminPassword: "your-secure-grafana-password"  # 實際使用 Secret
  
  # 持久化儲存（保存儀表板設定）
  persistence:
    enabled: true
    size: 10Gi
    storageClassName: longhorn
  
  # 預設安裝常用 Dashboard
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
      - name: default
        orgId: 1
        folder: ""
        type: file
        disableDeletion: false
        editable: true
        options:
          path: /var/lib/grafana/dashboards/default

  # 從 Grafana.com 自動匯入 Dashboard
  dashboards:
    default:
      # Node Exporter Full（ID: 1860）
      node-exporter-full:
        gnetId: 1860
        revision: 37
        datasource: Prometheus
      # Kubernetes Cluster Overview（ID: 7249）
      k8s-cluster:
        gnetId: 7249
        revision: 1
        datasource: Prometheus
      # Nginx Ingress Controller（ID: 9614）
      nginx-ingress:
        gnetId: 9614
        revision: 1
        datasource: Prometheus
      # ArgoCD（ID: 14584）
      argocd:
        gnetId: 14584
        revision: 1
        datasource: Prometheus
  
  # SMTP 設定（用於 Email 告警）
  grafana.ini:
    smtp:
      enabled: true
      host: smtp.gmail.com:465
      user: monitoring@company.com
      password: "smtp-password"
      from_address: monitoring@company.com
      from_name: Grafana Alerts
  
  # 資源設定
  resources:
    requests:
      cpu: 200m
      memory: 256Mi
    limits:
      cpu: 1
      memory: 1Gi

# Prometheus 設定
prometheus:
  prometheusSpec:
    # 資料保存時間
    retention: 30d
    # 資料保存大小上限
    retentionSize: "50GB"
    
    # 持久化儲存
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: longhorn
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi
    
    # 資源設定
    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: 2
        memory: 8Gi
    
    # 允許從所有 namespace 抓取 ServiceMonitor
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    ruleSelectorNilUsesHelmValues: false
    
    # 外部 Alertmanager URL（可選）
    externalUrl: "https://prometheus.company.com"
    
    # 額外抓取設定（靜態目標）
    additionalScrapeConfigs:
    - job_name: 'blackbox-http'
      metrics_path: /probe
      params:
        module: [http_2xx]
      static_configs:
      - targets:
        - https://api.company.com/health
        - https://app.company.com
      relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter.monitoring:9115

# Alertmanager 設定
alertmanager:
  alertmanagerSpec:
    retention: 120h
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: longhorn
          resources:
            requests:
              storage: 10Gi
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 512Mi

# Node Exporter 設定
nodeExporter:
  enabled: true

# kube-state-metrics 設定
kubeStateMetrics:
  enabled: true

# 關閉不需要的元件（按需調整）
kubeEtcd:
  enabled: true
  endpoints:
  - 192.168.1.10  # master 節點 IP
  - 192.168.1.11
  - 192.168.1.12
```

### 1.3 Alertmanager 路由設定（Slack 整合）

```yaml
---
# alertmanager-config.yaml（透過 Secret 掛載）
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-kube-prometheus-stack
  namespace: monitoring
stringData:
  alertmanager.yaml: |
    global:
      # 預設的告警解除後等待時間
      resolve_timeout: 5m
      # Slack 設定
      slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'

    # 告警範本
    templates:
    - '/etc/alertmanager/templates/*.tmpl'

    # 路由規則
    route:
      # 預設接收者
      receiver: 'slack-default'
      # 告警分組（避免告警風暴）
      group_by: ['alertname', 'namespace', 'severity']
      # 分組等待時間（收集更多告警再一起發送）
      group_wait: 30s
      # 重複告警間隔
      repeat_interval: 4h
      # 子路由
      routes:
      # CRITICAL 告警 → 立即通知 + PagerDuty
      - match:
          severity: critical
        receiver: 'pagerduty-critical'
        group_wait: 10s
        repeat_interval: 1h
        continue: true  # 繼續匹配其他路由（也發到 Slack）
      
      # WARNING 告警 → Slack #alerts channel
      - match:
          severity: warning
        receiver: 'slack-warning'
        group_wait: 1m
        repeat_interval: 4h
      
      # 特定 namespace（production）的告警 → 專屬頻道
      - match:
          namespace: production
        receiver: 'slack-production'
        group_wait: 30s
      
      # K8s 系統告警 → 不同頻道
      - match_re:
          alertname: (KubeNode.*|KubeController.*|KubeScheduler.*)
        receiver: 'slack-k8s-infra'

    receivers:
    # 預設接收者
    - name: 'slack-default'
      slack_configs:
      - channel: '#k8s-alerts'
        send_resolved: true
        title: '[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .CommonLabels.alertname }}'
        text: |
          *告警摘要：* {{ .CommonAnnotations.summary }}
          *詳細說明：* {{ .CommonAnnotations.description }}
          *嚴重程度：* {{ .CommonLabels.severity }}
          *命名空間：* {{ .CommonLabels.namespace }}
          *觸發時間：* {{ .CommonLabels.startsAt }}
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'

    - name: 'slack-warning'
      slack_configs:
      - channel: '#k8s-warnings'
        send_resolved: true
        title: '[WARNING] {{ .CommonLabels.alertname }}'
        text: |
          *摘要：* {{ .CommonAnnotations.summary }}
          *命名空間：* {{ .CommonLabels.namespace }}
          *詳細：* {{ .CommonAnnotations.description }}

    - name: 'slack-production'
      slack_configs:
      - channel: '#production-alerts'
        send_resolved: true
        title: '[PRODUCTION {{ .Status | toUpper }}] {{ .CommonLabels.alertname }}'
        text: |
          *摘要：* {{ .CommonAnnotations.summary }}
          *說明：* {{ .CommonAnnotations.description }}
          {{ if .CommonAnnotations.runbook_url }}*Runbook：* {{ .CommonAnnotations.runbook_url }}{{ end }}

    - name: 'slack-k8s-infra'
      slack_configs:
      - channel: '#k8s-infrastructure'
        send_resolved: true

    - name: 'pagerduty-critical'
      pagerduty_configs:
      - routing_key: 'your-pagerduty-routing-key'
        severity: 'critical'
        description: '{{ .CommonAnnotations.summary }}'

    # 靜默機制設定（維護視窗用）
    inhibit_rules:
    # 當 CRITICAL 告警觸發時，抑制相同告警的 WARNING
    - source_match:
        severity: 'critical'
      target_match:
        severity: 'warning'
      equal: ['alertname', 'namespace']
```

### 1.4 自訂 PrometheusRule 告警規則

```yaml
---
# custom-alerting-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: custom-k8s-rules
  namespace: monitoring
  labels:
    # 必須符合 Prometheus 的 ruleSelector
    release: kube-prometheus-stack
spec:
  groups:
  # Pod 相關告警
  - name: pod.rules
    interval: 30s
    rules:
    # Pod 重啟次數過多
    - alert: PodCrashLooping
      expr: |
        rate(kube_pod_container_status_restarts_total{namespace!="kube-system"}[15m]) * 60 * 15 > 3
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} 頻繁重啟"
        description: "命名空間 {{ $labels.namespace }} 中的 Pod {{ $labels.pod }} 在過去 15 分鐘重啟超過 3 次（目前：{{ $value | humanize }} 次）"
        runbook_url: "https://wiki.company.com/runbooks/pod-crashlooping"

    # Pod OOMKilled
    - alert: PodOOMKilled
      expr: |
        kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
      for: 0m  # 立即觸發
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} 因記憶體不足被殺死"
        description: "命名空間 {{ $labels.namespace }} 中的容器 {{ $labels.container }} 因 OOMKilled 重啟"

    # Pod 長時間 Pending
    - alert: PodPending
      expr: |
        kube_pod_status_phase{phase="Pending"} == 1
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} 長時間處於 Pending 狀態"
        description: "命名空間 {{ $labels.namespace }} 中的 Pod {{ $labels.pod }} 已 Pending 超過 10 分鐘"

  # 節點相關告警
  - name: node.rules
    rules:
    # 節點 CPU 使用率過高
    - alert: NodeCPUHigh
      expr: |
        100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "節點 {{ $labels.instance }} CPU 使用率過高"
        description: "節點 CPU 使用率為 {{ $value | humanize }}%，已超過 85%"

    # 節點記憶體使用率過高
    - alert: NodeMemoryHigh
      expr: |
        (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "節點 {{ $labels.instance }} 記憶體使用率過高"
        description: "節點記憶體使用率為 {{ $value | humanize }}%"

    # 節點磁碟空間不足
    - alert: NodeDiskSpaceLow
      expr: |
        (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 15
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "節點 {{ $labels.instance }} 磁碟空間不足"
        description: "節點 {{ $labels.instance }} 的根目錄可用空間僅剩 {{ $value | humanize }}%"

    # 節點離線
    - alert: NodeNotReady
      expr: kube_node_status_condition{condition="Ready",status="true"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "節點 {{ $labels.node }} 不可用"
        description: "K8s 節點 {{ $labels.node }} 已超過 5 分鐘處於 NotReady 狀態"

  # 應用程式 SLA 告警
  - name: application.rules
    rules:
    # HTTP 錯誤率過高（5xx）
    - alert: HighHTTPErrorRate
      expr: |
        sum(rate(http_requests_total{status=~"5..", namespace="production"}[5m]))
          / sum(rate(http_requests_total{namespace="production"}[5m])) > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "production namespace HTTP 5xx 錯誤率過高"
        description: "過去 5 分鐘的 HTTP 5xx 錯誤率為 {{ $value | humanizePercentage }}，超過 5%"

    # API P95 延遲過高
    - alert: HighAPILatency
      expr: |
        histogram_quantile(0.95,
          sum(rate(http_request_duration_seconds_bucket{namespace="production"}[5m])) by (le, service)
        ) > 1
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "服務 {{ $labels.service }} P95 延遲過高"
        description: "服務 {{ $labels.service }} 的 P95 延遲為 {{ $value | humanizeDuration }}，超過 1 秒"

  # Deployment 相關
  - name: deployment.rules
    rules:
    # Deployment 副本不足
    - alert: DeploymentReplicasMismatch
      expr: |
        kube_deployment_spec_replicas != kube_deployment_status_ready_replicas
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Deployment {{ $labels.deployment }} 副本數不符"
        description: "命名空間 {{ $labels.namespace }} 中的 Deployment {{ $labels.deployment }} 期望 {{ $value }} 個副本但實際可用數不符"

    # HPA 達到最大副本數
    - alert: HPAAtMaxReplicas
      expr: |
        kube_horizontalpodautoscaler_status_current_replicas
          == kube_horizontalpodautoscaler_spec_max_replicas
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "HPA {{ $labels.horizontalpodautoscaler }} 已達最大副本數"
        description: "HPA 已擴容到最大值 {{ $value }}，可能需要增加 maxReplicas"
```

---

## 2. 日誌管理（Loki Stack）

### 2.1 架構說明

```
Pod logs → Promtail (DaemonSet, 收集) → Loki (儲存) → Grafana (查詢/展示)
```

**Loki 與 ELK Stack 比較**

| 比較項目       | Loki                        | ELK Stack               |
|----------------|-----------------------------|-------------------------|
| 資源使用       | 低（不建索引內容）           | 高（全文索引）           |
| 查詢速度       | 中（依 label 過濾）          | 快（全文搜尋）           |
| 設定複雜度     | 低                           | 高                       |
| 成本           | 低                           | 高                       |
| 適用場景       | 基於 label 的日誌查詢        | 全文搜尋、複雜分析       |
| 與 Grafana 整合| 原生整合                     | 需要額外設定             |

### 2.2 Loki + Promtail 安裝

```bash
# 加入 Grafana Helm repo（Loki 由 Grafana Labs 維護）
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# 安裝 Loki Stack（Loki + Promtail）
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --values loki-values.yaml
```

**loki-values.yaml（完整設定）**

```yaml
# loki-values.yaml

loki:
  enabled: true
  # Loki 設定
  config:
    auth_enabled: false  # 單租戶模式
    
    server:
      http_listen_port: 3100
      grpc_listen_port: 9095
    
    common:
      path_prefix: /data/loki
      storage:
        filesystem:
          chunks_directory: /data/loki/chunks
          rules_directory: /data/loki/rules
      replication_factor: 1
    
    # 攝取設定
    ingester:
      wal:
        dir: /data/loki/wal
        enabled: true
      lifecycler:
        ring:
          replication_factor: 1
    
    # 資料保存設定
    compactor:
      retention_enabled: true
      retention_delete_delay: 2h
    
    limits_config:
      retention_period: 30d  # 保留 30 天
      # 速率限制
      ingestion_rate_mb: 16
      ingestion_burst_size_mb: 32
      per_stream_rate_limit: 5MB
      per_stream_rate_limit_burst: 15MB
      # 查詢限制
      max_query_series: 10000
      max_query_lookback: 90d
      max_entries_limit_per_query: 50000
    
    # 儲存設定（本地儲存，生產建議使用 S3/MinIO）
    schema_config:
      configs:
      - from: 2024-01-01
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: index_
          period: 24h
    
    # 查詢設定
    query_range:
      results_cache:
        cache:
          embedded_cache:
            enabled: true
            max_size_mb: 100
  
  # 持久化儲存
  persistence:
    enabled: true
    size: 50Gi
    storageClassName: longhorn
  
  resources:
    requests:
      cpu: 200m
      memory: 512Mi
    limits:
      cpu: 1
      memory: 2Gi

# Promtail 設定（DaemonSet，在每個節點收集日誌）
promtail:
  enabled: true
  config:
    # 日誌收集設定
    scrape_configs:
    # 收集 K8s Pod 日誌
    - job_name: kubernetes-pods
      pipeline_stages:
      # 解析 Docker/CRI-O 日誌格式
      - cri: {}
      # 解析 JSON 格式的應用程式日誌
      - json:
          expressions:
            level: level
            timestamp: time
            message: msg
            trace_id: trace_id
      # 提取 log level 作為 label
      - labels:
          level:
      # 過濾敏感資訊（如 password）
      - replace:
          expression: '(password|passwd|secret|token)=[^\s]+'
          replace: '${1}=REDACTED'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      # 只收集有特定 annotation 的 Pod（可選）
      # - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      #   action: keep
      #   regex: true
      - source_labels: [__meta_kubernetes_pod_node_name]
        action: replace
        target_label: node_name
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: pod
      - source_labels: [__meta_kubernetes_pod_container_name]
        action: replace
        target_label: container
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: replace
        target_label: app
      - replacement: /var/log/pods/*$1/*.log
        separator: /
        source_labels:
        - __meta_kubernetes_pod_uid
        - __meta_kubernetes_pod_container_name
        target_label: __path__
    
    # 收集 systemd journal（節點系統日誌）
    - job_name: systemd
      journal:
        max_age: 12h
        labels:
          job: systemd-journal
      pipeline_stages:
      - match:
          selector: '{job="systemd-journal"}'
          stages:
          - labels:
              unit: _SYSTEMD_UNIT
  
  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 200m
      memory: 256Mi

# 不重複安裝 Grafana（已透過 kube-prometheus-stack 安裝）
grafana:
  enabled: false
```

### 2.3 Grafana Loki 資料源設定

```bash
# 透過 API 新增 Loki 資料源
curl -X POST \
  -H "Content-Type: application/json" \
  -u admin:password \
  http://grafana.monitoring:3000/api/datasources \
  -d '{
    "name": "Loki",
    "type": "loki",
    "url": "http://loki.monitoring:3100",
    "access": "proxy",
    "isDefault": false,
    "jsonData": {
      "maxLines": 1000,
      "derivedFields": [
        {
          "datasourceUid": "prometheus-uid",
          "matcherRegex": "trace_id=(\\w+)",
          "name": "TraceID",
          "url": "${__value.raw}"
        }
      ]
    }
  }'
```

或透過 ConfigMap 設定（建議方式）：

```yaml
---
# grafana-datasource-loki.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasource-loki
  namespace: monitoring
  labels:
    grafana_datasource: "1"  # Grafana sidecar 會自動載入
data:
  loki-datasource.yaml: |
    apiVersion: 1
    datasources:
    - name: Loki
      type: loki
      url: http://loki.monitoring:3100
      access: proxy
      isDefault: false
      jsonData:
        maxLines: 1000
        timeout: 60
```

### 2.4 LogQL 查詢語法說明

LogQL 是 Loki 的查詢語言，語法類似 PromQL。

**基礎選擇器（Log Stream Selector）**

```logql
# 查詢特定 namespace 的所有日誌
{namespace="production"}

# 查詢特定 app 的日誌
{namespace="production", app="api-server"}

# 查詢特定 Pod 的日誌
{namespace="production", pod="api-server-5d9f7d8b9-xkr2p"}
```

**日誌過濾器（Log Pipeline）**

```logql
# 包含特定字串
{namespace="production"} |= "error"

# 排除特定字串
{namespace="production"} != "healthcheck"

# 正規表示式過濾
{namespace="production"} |~ "ERROR|WARN"

# 解析 JSON 日誌並過濾欄位
{namespace="production"} 
  | json 
  | level="error"

# 解析 JSON 後格式化輸出
{namespace="production"} 
  | json 
  | level="error" 
  | line_format "{{.timestamp}} [{{.level}}] {{.message}}"
```

**指標查詢（Metric Queries）**

```logql
# 每分鐘錯誤日誌數量
count_over_time({namespace="production"} |= "error" [1m])

# 每個 Pod 的每分鐘請求數
sum by (pod) (
  count_over_time({namespace="production", app="api-server"}[1m])
)

# 日誌速率（每秒）
rate({namespace="production"} [5m])

# 解析 access log，統計 HTTP 500 錯誤率
sum(rate({namespace="production", app="nginx"} 
  | regex `status=(?P<status>\d+)` 
  | status = "500" [5m]))
/
sum(rate({namespace="production", app="nginx"}[5m]))
```

**實用查詢範例**

```logql
# 查詢最近 15 分鐘的所有 ERROR（包含上下文）
{namespace="production"} |= "ERROR" | json | level="error"

# 統計各服務的錯誤數量
sum by (app) (
  count_over_time(
    {namespace="production"} |= "error" [15m]
  )
)

# 追蹤特定 trace_id
{namespace="production"} 
  | json 
  | trace_id="abc123def456"

# 查詢慢請求（duration > 1000ms）
{namespace="production", app="api-server"} 
  | json 
  | duration_ms > 1000
  | line_format "慢請求：{{.method}} {{.path}} 耗時 {{.duration_ms}}ms"

# K8s 事件（系統日誌）
{job="systemd-journal", unit="kubelet.service"} 
  |= "Error"
```

### 2.5 結構化日誌最佳實踐

```go
// Go 應用程式結構化日誌（使用 zerolog）
package main

import (
    "net/http"
    "os"
    "time"

    "github.com/rs/zerolog"
    "github.com/rs/zerolog/log"
)

func init() {
    // 設定 JSON 格式輸出（方便 Loki 解析）
    zerolog.TimeFieldFormat = time.RFC3339Nano
    
    // 根據環境設定 log level
    level := os.Getenv("LOG_LEVEL")
    switch level {
    case "debug":
        zerolog.SetGlobalLevel(zerolog.DebugLevel)
    case "warn":
        zerolog.SetGlobalLevel(zerolog.WarnLevel)
    case "error":
        zerolog.SetGlobalLevel(zerolog.ErrorLevel)
    default:
        zerolog.SetGlobalLevel(zerolog.InfoLevel)
    }
}

// HTTP middleware 範例：記錄所有請求
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // 從 context 取得 trace_id
        traceID := r.Header.Get("X-Trace-ID")
        if traceID == "" {
            traceID = generateTraceID()
        }

        // 建立帶有請求資訊的 logger
        reqLogger := log.With().
            Str("trace_id", traceID).
            Str("method", r.Method).
            Str("path", r.URL.Path).
            Str("remote_ip", r.RemoteAddr).
            Logger()

        // 將 logger 放入 context
        ctx := reqLogger.WithContext(r.Context())

        // 執行下一個 handler
        rw := &responseWriter{w, http.StatusOK}
        next.ServeHTTP(rw, r.WithContext(ctx))

        // 記錄請求完成
        duration := time.Since(start)
        reqLogger.Info().
            Int("status", rw.status).
            Dur("duration_ms", duration).
            Int64("bytes_sent", rw.bytesWritten).
            Msg("HTTP request completed")
    })
}
```

---

## 3. 分散式追蹤

### 3.1 OpenTelemetry 概念說明

OpenTelemetry (OTel) 是業界標準的可觀測性框架，支援 Traces、Metrics 和 Logs 三個信號。

```
應用程式（SDK）→ OpenTelemetry Collector → 後端（Tempo/Jaeger/Zipkin）
                                         → Prometheus（metrics）
                                         → Loki（logs）
```

**核心概念**

```
Trace（追蹤）：一個請求從開始到結束的完整路徑
Span（跨度）：Trace 的最小單位，代表一個操作
Context Propagation：跨服務傳遞 trace 上下文（W3C TraceContext header）
Instrumentation：在程式中加入 OTel 收集程式碼
```

### 3.2 OpenTelemetry Collector 部署

```yaml
---
# otel-collector.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: monitoring
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
      # 從 Prometheus 收集指標
      prometheus:
        config:
          scrape_configs:
          - job_name: otel-collector
            static_configs:
            - targets: ['0.0.0.0:8888']

    processors:
      # 批次處理（提升效能）
      batch:
        send_batch_size: 10000
        timeout: 10s
      # 記憶體限制（防止 OOM）
      memory_limiter:
        check_interval: 1s
        limit_mib: 512
        spike_limit_mib: 128
      # 加入 K8s 資源屬性
      k8sattributes:
        auth_type: serviceAccount
        passthrough: false
        extract:
          metadata:
          - k8s.pod.name
          - k8s.pod.uid
          - k8s.deployment.name
          - k8s.namespace.name
          - k8s.node.name

    exporters:
      # 輸出到 Tempo（追蹤）
      otlp/tempo:
        endpoint: http://tempo.monitoring:4317
        tls:
          insecure: true
      # 輸出到 Prometheus（指標）
      prometheusremotewrite:
        endpoint: http://prometheus-operated.monitoring:9090/api/v1/write
      # Debug 輸出（開發時使用）
      # debug:
      #   verbosity: detailed

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, k8sattributes, batch]
          exporters: [otlp/tempo]
        metrics:
          receivers: [otlp, prometheus]
          processors: [memory_limiter, batch]
          exporters: [prometheusremotewrite]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  replicas: 2
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      serviceAccountName: otel-collector
      containers:
      - name: otel-collector
        image: otel/opentelemetry-collector-contrib:0.96.0
        args: ["--config=/conf/config.yaml"]
        ports:
        - containerPort: 4317  # OTLP gRPC
          name: otlp-grpc
        - containerPort: 4318  # OTLP HTTP
          name: otlp-http
        - containerPort: 8888  # Prometheus metrics
          name: metrics
        volumeMounts:
        - name: config
          mountPath: /conf
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 1
            memory: 1Gi
      volumes:
      - name: config
        configMap:
          name: otel-collector-config
```

### 3.3 Tempo 安裝與設定

```bash
# 安裝 Grafana Tempo
helm install tempo grafana/tempo \
  --namespace monitoring \
  --values tempo-values.yaml
```

```yaml
# tempo-values.yaml
tempo:
  storage:
    trace:
      backend: local
      local:
        path: /var/tempo/traces
      wal:
        path: /var/tempo/wal
  
  retention: 168h  # 保留 7 天

  # 資源設定
  resources:
    requests:
      cpu: 200m
      memory: 512Mi
    limits:
      cpu: 1
      memory: 2Gi

persistence:
  enabled: true
  size: 50Gi
  storageClassName: longhorn
```

---

## 4. 應用程式監控最佳實踐

### 4.1 Go 應用程式加入 Prometheus Metrics

```go
// metrics.go - Prometheus metrics 定義
package metrics

import (
    "net/http"
    "time"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    // HTTP 請求計數器（用於計算 RPS 和錯誤率）
    HTTPRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Namespace: "myapp",
            Name:      "http_requests_total",
            Help:      "HTTP 請求總數",
        },
        []string{"method", "path", "status_code"},
    )

    // HTTP 請求延遲直方圖（用於計算 P50/P95/P99）
    HTTPRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Namespace: "myapp",
            Name:      "http_request_duration_seconds",
            Help:      "HTTP 請求處理時間",
            // 桶的邊界（單位：秒）
            Buckets: []float64{0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0},
        },
        []string{"method", "path"},
    )

    // 當前活躍請求數（Gauge）
    HTTPRequestsInFlight = promauto.NewGauge(
        prometheus.GaugeOpts{
            Namespace: "myapp",
            Name:      "http_requests_in_flight",
            Help:      "目前正在處理的 HTTP 請求數",
        },
    )

    // 資料庫連線池狀態
    DBConnectionsActive = promauto.NewGauge(
        prometheus.GaugeOpts{
            Namespace: "myapp",
            Name:      "db_connections_active",
            Help:      "目前活躍的資料庫連線數",
        },
    )

    DBConnectionsIdle = promauto.NewGauge(
        prometheus.GaugeOpts{
            Namespace: "myapp",
            Name:      "db_connections_idle",
            Help:      "目前閒置的資料庫連線數",
        },
    )

    // 業務指標範例：訂單計數
    OrdersCreatedTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Namespace: "myapp",
            Name:      "orders_created_total",
            Help:      "建立的訂單總數",
        },
        []string{"status", "product_category"},
    )

    // 業務指標範例：訂單金額分布
    OrderAmount = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Namespace: "myapp",
            Name:      "order_amount_dollars",
            Help:      "訂單金額分布（美元）",
            Buckets:   prometheus.LinearBuckets(0, 50, 20), // 0 到 1000，每 50 一個桶
        },
        []string{"product_category"},
    )

    // 快取命中率
    CacheHitsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Namespace: "myapp",
            Name:      "cache_hits_total",
            Help:      "快取命中次數",
        },
        []string{"cache_name", "result"}, // result: hit / miss
    )
)

// MetricsMiddleware 收集 HTTP 指標的中間件
func MetricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // 追蹤進行中的請求數
        HTTPRequestsInFlight.Inc()
        defer HTTPRequestsInFlight.Dec()

        // 使用 responseWriter 包裝以取得 status code
        rw := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}
        next.ServeHTTP(rw, r)

        // 記錄指標
        duration := time.Since(start).Seconds()
        statusCode := http.StatusText(rw.statusCode)
        
        // 路徑正規化（避免 cardinality 爆炸）
        path := normalizePath(r.URL.Path)
        
        HTTPRequestsTotal.WithLabelValues(
            r.Method, path, statusCode,
        ).Inc()
        
        HTTPRequestDuration.WithLabelValues(
            r.Method, path,
        ).Observe(duration)
    })
}

// normalizePath 將動態路徑正規化，避免每個 ID 都成為獨立 label
// 例：/api/v1/users/12345 → /api/v1/users/:id
func normalizePath(path string) string {
    // 使用正規表示式替換 UUID 和數字 ID
    // 實際實作建議使用路由框架提供的功能（如 chi.RouteContext）
    return path
}

// 設定 /metrics 端點
func SetupMetrics(mux *http.ServeMux) {
    mux.Handle("/metrics", promhttp.Handler())
}
```

**Deployment 設定 metrics scraping**

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  template:
    metadata:
      labels:
        app: myapp
      annotations:
        # 告訴 Prometheus 要抓取這個 Pod 的 metrics
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: myapp
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 9090
          name: metrics  # 也可以用獨立 port 給 metrics
---
# 使用 ServiceMonitor（推薦方式）
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  namespace: monitoring
  labels:
    release: kube-prometheus-stack  # 必須符合 Prometheus 的 serviceMonitorSelector
spec:
  namespaceSelector:
    matchNames:
    - production
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
    scrapeTimeout: 10s
```

### 4.2 Python 應用程式加入 Prometheus Metrics

```python
# metrics.py - Python 應用程式 metrics（Flask 範例）
from prometheus_client import Counter, Histogram, Gauge, generate_latest, CONTENT_TYPE_LATEST
from functools import wraps
import time
from flask import Flask, request, Response

app = Flask(__name__)

# 定義 metrics
http_requests_total = Counter(
    'myapp_http_requests_total',
    'HTTP 請求總數',
    ['method', 'endpoint', 'status_code']
)

http_request_duration = Histogram(
    'myapp_http_request_duration_seconds',
    'HTTP 請求延遲',
    ['method', 'endpoint'],
    buckets=[0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
)

active_requests = Gauge(
    'myapp_active_requests',
    '目前進行中的請求數'
)

# 業務指標
orders_processed = Counter(
    'myapp_orders_processed_total',
    '已處理的訂單數',
    ['status']
)

queue_depth = Gauge(
    'myapp_queue_depth',
    '佇列深度',
    ['queue_name']
)

def monitor_requests(f):
    """裝飾器：自動收集 HTTP 指標"""
    @wraps(f)
    def decorated_function(*args, **kwargs):
        active_requests.inc()
        start_time = time.time()
        
        try:
            response = f(*args, **kwargs)
            status_code = response.status_code if hasattr(response, 'status_code') else 200
        except Exception as e:
            status_code = 500
            raise
        finally:
            duration = time.time() - start_time
            endpoint = request.url_rule.rule if request.url_rule else request.path
            
            http_requests_total.labels(
                method=request.method,
                endpoint=endpoint,
                status_code=str(status_code)
            ).inc()
            
            http_request_duration.labels(
                method=request.method,
                endpoint=endpoint
            ).observe(duration)
            
            active_requests.dec()
        
        return response
    return decorated_function

@app.route('/metrics')
def metrics_endpoint():
    """Prometheus metrics 端點"""
    return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)

@app.route('/api/v1/orders', methods=['POST'])
@monitor_requests
def create_order():
    # 業務邏輯...
    try:
        # 處理訂單
        orders_processed.labels(status='success').inc()
        return {'order_id': '12345'}, 201
    except Exception as e:
        orders_processed.labels(status='error').inc()
        raise

# 背景工作：更新 Gauge 類型 metrics
import threading

def update_queue_metrics():
    while True:
        # 從 Redis/資料庫查詢佇列深度
        queue_depth.labels(queue_name='email_queue').set(get_email_queue_depth())
        queue_depth.labels(queue_name='notification_queue').set(get_notification_queue_depth())
        time.sleep(15)

threading.Thread(target=update_queue_metrics, daemon=True).start()
```

### 4.3 自訂 Grafana Dashboard 建議

**關鍵監控面板（RED Method）**

```
RED Method（服務監控的黃金標準）：
R - Rate（速率）：每秒請求數（RPS）
E - Errors（錯誤）：錯誤率（%）
D - Duration（時間）：請求延遲分布（P50/P95/P99）
```

**PromQL 查詢範例（可直接用於 Grafana Panel）**

```promql
# RPS（每秒請求數）
sum(rate(myapp_http_requests_total{namespace="production"}[5m]))

# 每個 endpoint 的 RPS
sum by (endpoint) (
  rate(myapp_http_requests_total{namespace="production"}[5m])
)

# 錯誤率（4xx + 5xx）
sum(rate(myapp_http_requests_total{namespace="production", status_code=~"[45].."}[5m]))
  / sum(rate(myapp_http_requests_total{namespace="production"}[5m])) * 100

# P99 延遲
histogram_quantile(0.99,
  sum by (le, endpoint) (
    rate(myapp_http_request_duration_seconds_bucket{namespace="production"}[5m])
  )
)

# 活躍請求數（Gauge）
sum(myapp_active_requests{namespace="production"})

# 訂單成功率
sum(rate(myapp_orders_processed_total{namespace="production", status="success"}[5m]))
  / sum(rate(myapp_orders_processed_total{namespace="production"}[5m])) * 100
```

### 4.4 Grafana Dashboard as Code（GitOps 管理）

```yaml
---
# grafana-dashboard-configmap.yaml
# 將 Dashboard JSON 存在 ConfigMap，Grafana sidecar 自動載入
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-myapp
  namespace: monitoring
  labels:
    grafana_dashboard: "1"  # Grafana sidecar 識別標籤
data:
  myapp-dashboard.json: |
    {
      "title": "MyApp Production Dashboard",
      "uid": "myapp-prod",
      "panels": [
        {
          "title": "Request Rate (RPS)",
          "type": "graph",
          "targets": [
            {
              "expr": "sum(rate(myapp_http_requests_total{namespace=\"production\"}[5m]))",
              "legendFormat": "Total RPS"
            }
          ]
        },
        {
          "title": "Error Rate (%)",
          "type": "singlestat",
          "targets": [
            {
              "expr": "sum(rate(myapp_http_requests_total{namespace=\"production\",status_code=~\"[45]..\"}[5m])) / sum(rate(myapp_http_requests_total{namespace=\"production\"}[5m])) * 100"
            }
          ],
          "thresholds": "1,5",
          "colors": ["green", "orange", "red"]
        }
      ],
      "refresh": "30s",
      "time": {
        "from": "now-1h",
        "to": "now"
      }
    }
```

---

## 附錄：監控最佳實踐清單

```
□ Metrics 覆蓋度
  □ 所有服務暴露 /metrics 端點
  □ 涵蓋 RED（Rate/Errors/Duration）三個維度
  □ 業務關鍵指標（訂單成功率、用戶活躍數等）

□ Alerting 品質
  □ 告警規則基於 SLO/SLA 制定
  □ 告警包含 runbook_url（說明如何處理）
  □ 避免 alert fatigue（太多無意義告警）
  □ 每個 CRITICAL 告警都有明確的 on-call 流程

□ Logging 規範
  □ 所有服務使用 JSON 結構化日誌
  □ 包含 trace_id 方便跨服務追蹤
  □ 敏感資訊（密碼/token）不寫入日誌
  □ 日誌保留政策符合法規要求（例：90天）

□ Dashboard 維護
  □ 每個服務有對應的 Grafana Dashboard
  □ Dashboard 存放在 Git（as code）
  □ 包含 SLO Dashboard（讓 PM 也能查看）

□ 可觀測性成熟度
  □ Metrics（✓ Prometheus）
  □ Logs（✓ Loki）
  □ Traces（← 下一步：OpenTelemetry + Tempo）
  □ 關聯分析（Trace ID 串連 Logs 和 Traces）
```
