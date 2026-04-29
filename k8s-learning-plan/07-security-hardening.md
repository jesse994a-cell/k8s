# Kubernetes 資安強化完整指南

> 環境：3 master + 3 worker，Ubuntu 24.04，GitLab + ArgoCD CI/CD

---

## 目錄

1. [資安框架概覽](#1-資安框架概覽)
2. [RBAC 角色型存取控制](#2-rbac-角色型存取控制)
3. [Network Policy 網路政策](#3-network-policy-網路政策)
4. [Pod Security Standards](#4-pod-security-standards)
5. [Falco 威脅偵測](#5-falco-威脅偵測)
6. [Trivy 映像掃描](#6-trivy-映像掃描)
7. [HashiCorp Vault 機密管理](#7-hashicorp-vault-機密管理)
8. [稽核日誌](#8-稽核日誌)

---

## 1. 資安框架概覽

### 1.1 K8s 資安縱深防禦：4C 模型

4C 模型是 Kubernetes 官方推薦的資安分層防禦策略，每一層都需要獨立保護，任何一層的失守都可能危及整體安全。

```
┌─────────────────────────────────────────────────────┐
│  Cloud（雲端 / 基礎設施層）                           │
│  ┌───────────────────────────────────────────────┐  │
│  │  Cluster（叢集層）                             │  │
│  │  ┌───────────────────────────────────────┐   │  │
│  │  │  Container（容器層）                   │   │  │
│  │  │  ┌───────────────────────────────┐   │   │  │
│  │  │  │  Code（程式碼層）              │   │   │  │
│  │  │  └───────────────────────────────┘   │   │  │
│  │  └───────────────────────────────────────┘   │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

| 層級      | 範疇                        | 防護措施                                      |
|-----------|-----------------------------|-----------------------------------------------|
| Cloud     | IaaS、網路、實體機           | firewall、IAM、SSH key 管理、節點 OS 強化      |
| Cluster   | K8s API、etcd、node         | RBAC、NetworkPolicy、Audit log、PSS            |
| Container | 映像、runtime、namespace     | Trivy、Falco、seccomp、AppArmor                |
| Code      | 應用程式、套件、配置檔       | SAST、依賴掃描、Vault 機密管理                  |

### 1.2 CIS Kubernetes Benchmark 重點

CIS (Center for Internet Security) 提供業界標準的 K8s 安全基準測試。

**API Server 重要設定（/etc/kubernetes/manifests/kube-apiserver.yaml）**

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    # 停用匿名認證
    - --anonymous-auth=false
    # 啟用 RBAC 與 Node 授權
    - --authorization-mode=Node,RBAC
    # 停用不安全的 HTTP port
    - --insecure-port=0
    # 啟用稽核日誌
    - --audit-log-path=/var/log/kubernetes/audit.log
    - --audit-log-maxage=30
    - --audit-log-maxbackup=10
    - --audit-log-maxsize=100
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    # 啟用 Admission Controller
    - --enable-admission-plugins=NodeRestriction,PodSecurity,ResourceQuota,LimitRanger
    # TLS 設定
    - --tls-min-version=VersionTLS12
    - --tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
    # 停用 profiling（避免資訊洩漏）
    - --profiling=false
    # 啟用 ServiceAccount token 自動掛載限制
    - --service-account-lookup=true
```

**etcd 重要設定**

```yaml
spec:
  containers:
  - command:
    - etcd
    # 強制 TLS 客戶端認證
    - --client-cert-auth=true
    - --peer-client-cert-auth=true
    # 加密靜態資料（配合 EncryptionConfig）
    - --encryption-provider-config=/etc/kubernetes/enc/encrypt.yaml
```

**執行 CIS 掃描工具 kube-bench**

```bash
# 在 master 節點執行
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job-master.yaml
kubectl logs job/kube-bench-master

# 在 worker 節點執行
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job-node.yaml
kubectl logs job/kube-bench-node
```

---

## 2. RBAC 角色型存取控制

### 2.1 核心概念說明

| 資源             | Namespace 範疇 | 說明                                     |
|------------------|----------------|------------------------------------------|
| Role             | 是             | 定義單一 namespace 內的權限              |
| ClusterRole      | 否             | 定義跨 namespace 或叢集層級的權限        |
| RoleBinding      | 是             | 將 Role 綁定給 User/Group/ServiceAccount |
| ClusterRoleBinding | 否           | 將 ClusterRole 綁定給主體，全叢集有效    |

**RBAC 決策流程**

```
請求 → Authentication（誰？）→ Authorization（可以嗎？）→ AdmissionControl（允許嗎？）
```

### 2.2 最小權限原則實踐

**原則一：明確列出 verbs，禁止使用萬用字元**
```yaml
# 錯誤示範 - 過度授權
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]

# 正確示範
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
```

**原則二：定期審查綁定關係**
```bash
# 列出所有 ClusterRoleBinding
kubectl get clusterrolebindings -o wide

# 找出 cluster-admin 綁定
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name=="cluster-admin") | .metadata.name, .subjects'

# 找出某個 ServiceAccount 的所有權限
kubectl auth can-i --list --as=system:serviceaccount:default:myapp
```

### 2.3 完整 YAML 範例

**開發者 Role（dev-role.yaml）**

```yaml
---
# 定義 Role：開發者在 dev namespace 的權限
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: developer-role
rules:
# 允許查看 Pod 和 log
- apiGroups: [""]
  resources: ["pods", "pods/log", "pods/exec"]
  verbs: ["get", "list", "watch", "create", "delete"]
# 允許查看和管理 Deployment
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
# 允許查看 Service
- apiGroups: [""]
  resources: ["services", "endpoints"]
  verbs: ["get", "list", "watch"]
# 允許查看 ConfigMap（不允許寫入 Secret）
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
# 允許查看事件（troubleshooting 用）
- apiGroups: [""]
  resources: ["events"]
  verbs: ["get", "list", "watch"]
---
# RoleBinding：將 developer-role 綁定給開發者群組
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: development
subjects:
# 綁定整個開發者群組（透過 OIDC/LDAP 整合）
- kind: Group
  name: dev-team
  apiGroup: rbac.authorization.k8s.io
# 也可以綁定個別使用者
- kind: User
  name: alice@company.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

**CI/CD Robot ServiceAccount Role（cicd-role.yaml）**

```yaml
---
# ServiceAccount 給 GitLab Runner / ArgoCD 使用
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cicd-robot
  namespace: production
  annotations:
    description: "CI/CD pipeline service account - DO NOT give extra permissions"
---
# ClusterRole：CI/CD 需要的最小叢集權限
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cicd-deployer-role
rules:
# 部署應用程式所需
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets", "daemonsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
# 管理 Service 與 Ingress
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
# 滾動更新需要查看 Pod 狀態
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
# 管理 ConfigMap（不含 Secret）
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# 查看 ReplicaSet（用於確認 rollout 狀態）
- apiGroups: ["apps"]
  resources: ["replicasets"]
  verbs: ["get", "list", "watch"]
---
# ClusterRoleBinding：限定在特定 namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cicd-deployer-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: cicd-robot
  namespace: production
roleRef:
  kind: ClusterRole
  name: cicd-deployer-role
  apiGroup: rbac.authorization.k8s.io
```

**唯讀監控 ClusterRole（monitoring-role.yaml）**

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-viewer
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/metrics
  - nodes/proxy
  - pods
  - services
  - endpoints
  - namespaces
  resources: ["*"]
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics", "/metrics/cadvisor"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-monitoring
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: monitoring-viewer
  apiGroup: rbac.authorization.k8s.io
```

### 2.4 ServiceAccount 最佳實踐

```yaml
---
# 為每個應用建立專屬 ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
# 停用自動掛載 token（大多數 Pod 不需要呼叫 K8s API）
automountServiceAccountToken: false
---
# 在 Pod spec 中明確控制
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      serviceAccountName: myapp-sa
      # 明確設定不自動掛載
      automountServiceAccountToken: false
      containers:
      - name: myapp
        image: myapp:v1.0.0
```

### 2.5 RBAC 稽核方式

```bash
# 1. 檢視指定用戶的權限
kubectl auth can-i --list --as=developer@company.com

# 2. 測試特定操作
kubectl auth can-i delete pods --as=developer@company.com -n production

# 3. 使用 kubectl-who-can 插件（需另行安裝）
kubectl who-can delete pods -n production

# 4. 使用 rbac-tool 分析（開源工具）
# 安裝
curl -L https://github.com/alcideio/rbac-tool/releases/latest/download/rbac-tool_linux_amd64 -o rbac-tool
chmod +x rbac-tool && mv rbac-tool /usr/local/bin/

# 產生視覺化報告
rbac-tool viz --outformat dot | dot -Tpng > rbac-graph.png

# 5. 稽核 audit log 中的 RBAC 拒絕事件
grep '"verb":"create"' /var/log/kubernetes/audit.log | \
  jq 'select(.annotations["authorization.k8s.io/decision"]=="forbid")'
```

---

## 3. Network Policy 網路政策

### 3.1 Calico NetworkPolicy 說明

Kubernetes 原生 NetworkPolicy 功能需要 CNI 插件支援，Calico 提供比原生更豐富的功能：

| 功能           | K8s 原生 NetworkPolicy | Calico NetworkPolicy       |
|----------------|------------------------|----------------------------|
| Namespace 隔離 | 支援                   | 支援                       |
| CIDR 規則      | 支援                   | 支援（更豐富）              |
| DNS 名稱規則   | 不支援                 | 支援                       |
| 全域預設政策   | 不支援                 | 支援（GlobalNetworkPolicy） |
| 7 層規則       | 不支援                 | 支援（需 Istio 整合）       |
| 優先順序       | 不支援                 | 支援                       |

### 3.2 Default Deny All 策略

**強烈建議** 在每個 namespace 先設定 default deny，再逐步開放必要流量：

```yaml
---
# 預設封鎖所有 ingress 流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}  # 選取所有 Pod
  policyTypes:
  - Ingress
---
# 預設封鎖所有 egress 流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  # 允許 DNS 查詢（否則所有服務發現都會失敗）
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
```

### 3.3 完整三層架構隔離範例

**架構說明**

```
Internet → [Ingress Controller] → [Frontend] → [Backend API] → [Database]
                                                    ↓
                                              [Redis Cache]
```

**frontend-netpol.yaml**

```yaml
---
# 前端：允許從 Ingress Controller 進入，允許連線到 Backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # 只允許 Ingress Controller 的流量進入
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
      podSelector:
        matchLabels:
          app.kubernetes.io/name: ingress-nginx
    ports:
    - protocol: TCP
      port: 3000  # 前端應用程式 port
  egress:
  # 允許連線到後端 API
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 8080
  # 允許 DNS 查詢
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
  # 允許連線到外部 CDN（如需要）
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8
        - 172.16.0.0/12
        - 192.168.0.0/16
    ports:
    - protocol: TCP
      port: 443
```

**backend-netpol.yaml**

```yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # 只允許前端流量
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
  # 允許 Prometheus 爬取 metrics
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring
      podSelector:
        matchLabels:
          app: prometheus
    ports:
    - protocol: TCP
      port: 9090
  egress:
  # 允許連線到資料庫
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432  # PostgreSQL
  # 允許連線到 Redis
  - to:
    - podSelector:
        matchLabels:
          tier: cache
    ports:
    - protocol: TCP
      port: 6379
  # 允許 DNS
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
```

**database-netpol.yaml**

```yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # 只允許後端應用程式連線
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
  # 允許資料庫備份 job
  - from:
    - podSelector:
        matchLabels:
          app: db-backup
    ports:
    - protocol: TCP
      port: 5432
  egress:
  # 資料庫不需要主動發起連線（除了 DNS 和備份儲存）
  - ports:
    - port: 53
      protocol: UDP
  # 允許連線到備份儲存（如 S3 compatible）
  - to:
    - ipBlock:
        cidr: 10.100.0.0/16  # 內部備份儲存 IP
    ports:
    - protocol: TCP
      port: 9000
```

### 3.4 Namespace 隔離設計

```yaml
---
# 禁止不同 namespace 的 Pod 互相通訊
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-namespace
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  # 只允許同 namespace 的流量
  - from:
    - podSelector: {}
  # 允許 monitoring namespace 的監控
  - from:
    - namespaceSelector:
        matchLabels:
          purpose: monitoring
```

---

## 4. Pod Security Standards

### 4.1 三個安全層級說明

| 層級        | 說明                                   | 適用場景                        |
|-------------|----------------------------------------|---------------------------------|
| Privileged  | 完全不受限制                            | 系統級工具（Falco、CNI plugins）|
| Baseline    | 防止常見的權限提升，允許一般應用程式需求 | 一般應用程式                    |
| Restricted  | 最嚴格，遵循 Pod 硬化最佳實踐           | 安全敏感的應用程式              |

### 4.2 PodSecurityAdmission 設定

在 Namespace 層級透過 label 控制：

```bash
# 為 namespace 設定安全標準
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=v1.28 \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=v1.28 \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/audit-version=v1.28
```

```yaml
# namespace 設定範例
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # enforce：違規的 Pod 會被拒絕
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.28
    # warn：顯示警告但不拒絕
    pod-security.kubernetes.io/warn: restricted
    # audit：記錄到稽核日誌
    pod-security.kubernetes.io/audit: restricted
```

**全域設定（kube-apiserver）**

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
- --admission-control-config-file=/etc/kubernetes/admission-config.yaml
```

```yaml
# /etc/kubernetes/admission-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: baseline
      enforce-version: latest
      audit: restricted
      audit-version: latest
      warn: restricted
      warn-version: latest
    exemptions:
      usernames: []
      runtimeClasses: []
      namespaces:
      - kube-system
      - kube-public
      - falco      # Falco 需要 privileged
      - calico-system
```

### 4.3 SecurityContext 最佳實踐

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: secure-app
  template:
    metadata:
      labels:
        app: secure-app
    spec:
      # Pod 層級的安全設定
      securityContext:
        # 以非 root 使用者執行（UID 1000）
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        # 設定 supplemental groups
        fsGroup: 2000
        # 啟用 seccomp profile（限制 syscall）
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: app
        image: myapp:v1.0.0
        # 容器層級安全設定
        securityContext:
          # 禁止提升權限
          allowPrivilegeEscalation: false
          # 非 root
          runAsNonRoot: true
          runAsUser: 1000
          # 唯讀根目錄（強烈建議）
          readOnlyRootFilesystem: true
          # 移除所有 capabilities，只加入需要的
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE  # 如需綁定 1024 以下 port
        # 如需寫入，使用 emptyDir
        volumeMounts:
        - name: tmp-dir
          mountPath: /tmp
        - name: cache-dir
          mountPath: /app/cache
      volumes:
      - name: tmp-dir
        emptyDir: {}
      - name: cache-dir
        emptyDir:
          sizeLimit: 100Mi
      # 不自動掛載 SA token
      automountServiceAccountToken: false
```

---

## 5. Falco 威脅偵測

### 5.1 Falco 架構說明

Falco 透過 eBPF 或 kernel module 監控系統呼叫，即時偵測異常行為。

```
┌────────────────────────────────────────┐
│  Linux Kernel                          │
│  ┌──────────────────────────────────┐  │
│  │  syscall events (via eBPF probe) │  │
│  └──────────────┬───────────────────┘  │
└─────────────────┼──────────────────────┘
                  │
         ┌────────▼────────┐
         │  Falco Engine   │
         │  ┌───────────┐  │
         │  │   Rules   │  │
         │  │  Engine   │  │
         │  └─────┬─────┘  │
         └────────┼─────────┘
                  │ alerts
         ┌────────▼──────────────────┐
         │  Falco Sidekick           │
         │  (Slack/Webhook/PagerDuty)│
         └───────────────────────────┘
```

### 5.2 使用 Helm 安裝 Falco

```bash
# 加入 Falco Helm repository
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

# 建立 namespace
kubectl create namespace falco

# 安裝 Falco（使用 eBPF 驅動，適合現代 kernel）
helm install falco falcosecurity/falco \
  --namespace falco \
  --values falco-values.yaml
```

**falco-values.yaml（完整設定）**

```yaml
# falco-values.yaml
driver:
  kind: ebpf  # 推薦：不需要 kernel module，相容性更好

falco:
  # 啟用 JSON 輸出（方便後續解析）
  json_output: true
  json_include_output_property: true
  # 輸出到標準輸出（讓 log 收集器收集）
  log_stderr: true
  log_syslog: false
  # 輸出等級（emergency/alert/critical/error/warning/notice/info/debug）
  log_level: info
  # 規則優先等級（低於此等級的告警不輸出）
  priority: warning
  # 緩衝區設定（高流量環境）
  buffered_outputs: true
  outputs:
    rate: 1
    max_burst: 1000

# 啟用 Falco Sidekick
falcosidekick:
  enabled: true
  replicaCount: 2
  config:
    slack:
      webhookurl: "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
      channel: "#k8s-security-alerts"
      minimumpriority: "warning"
      messageformat: "長格式"
    webhook:
      address: "http://alertmanager.monitoring:9093/api/v2/alerts"
      minimumpriority: "error"
    # 自訂輸出到 Elasticsearch
    elasticsearch:
      hostport: "http://elasticsearch.logging:9200"
      index: "falco"
      minimumpriority: "notice"

# 自訂規則（掛載 ConfigMap）
customRules:
  custom-rules.yaml: |-
    # 偵測在 production namespace 執行 shell
    - rule: Shell Spawned in Production Container
      desc: 偵測在 production namespace 的 container 中執行互動式 shell
      condition: >
        spawned_process and
        container and
        k8s.ns.name = "production" and
        (proc.name = bash or proc.name = sh or proc.name = zsh) and
        interactive
      output: >
        互動式 shell 在 production container 中啟動
        (user=%user.name user_loginuid=%user.loginuid
        pod=%k8s.pod.name ns=%k8s.ns.name
        image=%container.image.repository:%container.image.tag
        shell=%proc.name parent=%proc.pname cmdline=%proc.cmdline)
      priority: CRITICAL
      tags: [shell, production, T1059]

    # 偵測讀取敏感檔案
    - rule: Read Sensitive File in Container
      desc: 偵測讀取 /etc/shadow、/etc/passwd 等敏感檔案
      condition: >
        open_read and
        container and
        (fd.name = /etc/shadow or
         fd.name = /etc/sudoers or
         fd.name startswith /etc/sudoers.d or
         fd.name = /root/.ssh/id_rsa) and
        not proc.name in (sshd, groupadd, useradd)
      output: >
        敏感檔案被讀取
        (user=%user.name
        pod=%k8s.pod.name ns=%k8s.ns.name
        file=%fd.name proc=%proc.name cmdline=%proc.cmdline)
      priority: WARNING
      tags: [sensitive_files, T1552]

    # 偵測 cryptomining（常見的容器逃逸後行為）
    - rule: Crypto Mining Activity
      desc: 偵測可能的加密貨幣挖礦活動
      condition: >
        spawned_process and
        container and
        (proc.name in (xmrig, minerd, cpuminer, cgminer) or
         proc.cmdline contains "stratum+tcp" or
         proc.cmdline contains "pool.minergate.com")
      output: >
        可能的加密貨幣挖礦活動
        (user=%user.name pod=%k8s.pod.name ns=%k8s.ns.name
        proc=%proc.name cmdline=%proc.cmdline)
      priority: CRITICAL
      tags: [cryptomining, T1496]

    # 偵測 Pod 中的網路掃描工具
    - rule: Network Scanner Tool Executed
      desc: 偵測 nmap/masscan 等掃描工具
      condition: >
        spawned_process and
        container and
        proc.name in (nmap, masscan, zmap, arp-scan)
      output: >
        網路掃描工具執行
        (user=%user.name pod=%k8s.pod.name ns=%k8s.ns.name
        proc=%proc.name cmdline=%proc.cmdline)
      priority: ERROR
      tags: [network_scan, T1046]

# Pod 設定（DaemonSet 需要特殊權限）
podSecurityContext:
  runAsNonRoot: false

containerSecurityContext:
  privileged: false
  capabilities:
    add:
    - SYS_PTRACE
    - SYS_ADMIN  # eBPF 需要

tolerations:
- effect: NoSchedule
  key: node-role.kubernetes.io/control-plane
  operator: Exists

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 1024Mi
```

### 5.3 Falco 告警查看與管理

```bash
# 查看 Falco 告警
kubectl logs -n falco -l app.kubernetes.io/name=falco -f

# 查看 Falco Sidekick 狀態
kubectl logs -n falco -l app.kubernetes.io/name=falco-sidekick -f

# 查看 Falco Sidekick UI（如有啟用）
kubectl port-forward -n falco svc/falco-falcosidekick-ui 2802:2802
```

---

## 6. Trivy 映像掃描

### 6.1 在 GitLab CI 中整合 Trivy

**完整 .gitlab-ci.yml 範例**

```yaml
# .gitlab-ci.yml
stages:
  - build
  - scan
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  IMAGE_NAME: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  # Trivy 設定
  TRIVY_NO_PROGRESS: "true"
  TRIVY_CACHE_DIR: ".trivycache/"
  # 嚴重等級門檻：CRITICAL 和 HIGH 會導致 pipeline 失敗
  TRIVY_SEVERITY: "CRITICAL,HIGH"
  TRIVY_EXIT_CODE: "1"
  # 忽略未修復的漏洞（可選，視安全政策而定）
  TRIVY_IGNORE_UNFIXED: "true"

build:
  stage: build
  image: docker:24
  services:
  - docker:24-dind
  script:
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  - docker build -t $IMAGE_NAME .
  - docker push $IMAGE_NAME
  artifacts:
    paths:
    - docker-image-id.txt

container-scanning:
  stage: scan
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  variables:
    # 掃描目標
    SCAN_IMAGE: $IMAGE_NAME
  cache:
    paths:
    - .trivycache/
  before_script:
  - trivy --version
  - trivy image --download-db-only
  script:
  # 輸出可讀格式供人工檢視
  - |
    trivy image \
      --cache-dir $TRIVY_CACHE_DIR \
      --severity $TRIVY_SEVERITY \
      --format table \
      --output trivy-report.txt \
      $SCAN_IMAGE
  # 輸出 JSON 格式供 GitLab Security Dashboard
  - |
    trivy image \
      --cache-dir $TRIVY_CACHE_DIR \
      --severity CRITICAL,HIGH,MEDIUM,LOW \
      --format json \
      --output trivy-report.json \
      $SCAN_IMAGE
  # 轉換為 GitLab 格式
  - |
    trivy image \
      --cache-dir $TRIVY_CACHE_DIR \
      --severity $TRIVY_SEVERITY \
      --format template \
      --template "@/contrib/gitlab.tpl" \
      --output gl-container-scanning-report.json \
      --exit-code $TRIVY_EXIT_CODE \
      $SCAN_IMAGE
  artifacts:
    when: always
    reports:
      container_scanning: gl-container-scanning-report.json
    paths:
    - trivy-report.txt
    - trivy-report.json
    - gl-container-scanning-report.json
    expire_in: 30 days
  allow_failure: false

# 掃描 K8s 設定檔（IaC 掃描）
k8s-config-scan:
  stage: scan
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
  - |
    trivy config \
      --severity HIGH,CRITICAL \
      --format table \
      ./k8s/
  artifacts:
    when: always
    paths:
    - trivy-config-report.txt
  allow_failure: true
```

### 6.2 Trivy Operator 部署（K8s 內持續掃描）

```bash
# 安裝 Trivy Operator
helm repo add aquasecurity https://aquasecurity.github.io/helm-charts/
helm repo update

helm install trivy-operator aquasecurity/trivy-operator \
  --namespace trivy-system \
  --create-namespace \
  --set="trivy.ignoreUnfixed=true" \
  --set="operator.scanJobTimeout=5m" \
  --set="operator.vulnerabilityReportsPlugin=Trivy" \
  --values trivy-operator-values.yaml
```

**trivy-operator-values.yaml**

```yaml
# trivy-operator-values.yaml
operator:
  # 自動掃描新建立的工作負載
  vulnerabilityScannerEnabled: true
  # 設定檔掃描
  configAuditScannerEnabled: true
  # 週期性重新掃描（防止新漏洞）
  vulnerabilityScannerReportTTL: "24h"

trivy:
  severity: "UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL"
  ignoreUnfixed: false
  # 使用私有 registry
  imageRef: "aquasec/trivy:0.48.0"
  resources:
    requests:
      cpu: 100m
      memory: 100Mi
    limits:
      cpu: 500m
      memory: 500Mi

# 設定掃描排程（每 6 小時）
scanner:
  jobTTLAfterFinished: 300
```

```bash
# 查看漏洞報告
kubectl get vulnerabilityreports -A
kubectl describe vulnerabilityreport <report-name> -n production

# 查看設定稽核報告
kubectl get configauditreports -A

# 匯出報告
kubectl get vulnerabilityreports -A -o json | \
  jq '.items[] | select(.report.summary.criticalCount > 0) | 
    {namespace: .metadata.namespace, name: .metadata.name, 
     critical: .report.summary.criticalCount}'
```

---

## 7. HashiCorp Vault 機密管理

### 7.1 在 K8s 上部署 Vault

```bash
# 加入 HashiCorp Helm repo
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# 建立 namespace
kubectl create namespace vault

# 安裝 Vault（HA 模式，3 副本）
helm install vault hashicorp/vault \
  --namespace vault \
  --values vault-values.yaml
```

**vault-values.yaml（HA 模式）**

```yaml
# vault-values.yaml
global:
  enabled: true
  tlsDisable: false  # 生產環境務必啟用 TLS

server:
  # HA 模式（3 副本，使用 Raft 共識）
  ha:
    enabled: true
    replicas: 3
    raft:
      enabled: true
      setNodeId: true
      config: |
        ui = true
        
        listener "tcp" {
          tls_disable = 0
          address = "[::]:8200"
          cluster_address = "[::]:8201"
          tls_cert_file = "/vault/userconfig/vault-tls/vault.crt"
          tls_key_file  = "/vault/userconfig/vault-tls/vault.key"
          tls_client_ca_file = "/vault/userconfig/vault-tls/vault.ca"
        }
        
        storage "raft" {
          path = "/vault/data"
          
          retry_join {
            leader_api_addr = "https://vault-0.vault-internal:8200"
            leader_ca_cert_file = "/vault/userconfig/vault-tls/vault.ca"
            leader_client_cert_file = "/vault/userconfig/vault-tls/vault.crt"
            leader_client_key_file = "/vault/userconfig/vault-tls/vault.key"
          }
          retry_join {
            leader_api_addr = "https://vault-1.vault-internal:8200"
            leader_ca_cert_file = "/vault/userconfig/vault-tls/vault.ca"
            leader_client_cert_file = "/vault/userconfig/vault-tls/vault.crt"
            leader_client_key_file = "/vault/userconfig/vault-tls/vault.key"
          }
          retry_join {
            leader_api_addr = "https://vault-2.vault-internal:8200"
            leader_ca_cert_file = "/vault/userconfig/vault-tls/vault.ca"
            leader_client_cert_file = "/vault/userconfig/vault-tls/vault.crt"
            leader_client_key_file = "/vault/userconfig/vault-tls/vault.key"
          }
        }
        
        service_registration "kubernetes" {}

  resources:
    requests:
      memory: 256Mi
      cpu: 250m
    limits:
      memory: 1Gi
      cpu: 1000m

  dataStorage:
    enabled: true
    size: 10Gi
    storageClass: longhorn  # 依環境調整

# 啟用 Vault Agent Injector
injector:
  enabled: true
  replicas: 2
  resources:
    requests:
      memory: 64Mi
      cpu: 50m
    limits:
      memory: 128Mi
      cpu: 250m

# Vault UI
ui:
  enabled: true
  serviceType: ClusterIP
```

### 7.2 初始化與解封 Vault

```bash
# 初始化 Vault（只需執行一次）
kubectl exec -n vault vault-0 -- vault operator init \
  -key-shares=5 \
  -key-threshold=3 \
  -format=json > vault-init.json

# 重要：安全保存 vault-init.json 中的 unseal keys 和 root token！
# 實際生產環境應使用 PGP 加密或 HSM

# 解封（需要 3/5 的 unseal key）
kubectl exec -n vault vault-0 -- vault operator unseal <UNSEAL_KEY_1>
kubectl exec -n vault vault-0 -- vault operator unseal <UNSEAL_KEY_2>
kubectl exec -n vault vault-0 -- vault operator unseal <UNSEAL_KEY_3>

# 驗證狀態
kubectl exec -n vault vault-0 -- vault status
```

### 7.3 設定 Kubernetes Auth Method

```bash
# 登入 Vault
export VAULT_TOKEN=$(cat vault-init.json | jq -r '.root_token')
kubectl exec -n vault vault-0 -- vault login $VAULT_TOKEN

# 啟用 Kubernetes auth method
kubectl exec -n vault vault-0 -- vault auth enable kubernetes

# 設定 Kubernetes auth（取得 K8s API 資訊）
KUBE_HOST=$(kubectl config view --raw --minify --flatten \
  --output='jsonpath={.clusters[].cluster.server}')
KUBE_CA=$(kubectl config view --raw --minify --flatten \
  --output='jsonpath={.clusters[].cluster.certificate-authority-data}' | base64 --decode)

kubectl exec -n vault vault-0 -- vault write auth/kubernetes/config \
  kubernetes_host="$KUBE_HOST" \
  kubernetes_ca_cert="$KUBE_CA" \
  disable_local_ca_jwt=false
```

### 7.4 建立 KV Secret Engine 與 Policy

```bash
# 啟用 KV v2 secret engine
kubectl exec -n vault vault-0 -- vault secrets enable -path=secret kv-v2

# 建立一些範例機密
kubectl exec -n vault vault-0 -- vault kv put secret/production/database \
  username="dbuser" \
  password="SuperSecretPassword123!" \
  host="postgres.production.svc.cluster.local" \
  port="5432"

kubectl exec -n vault vault-0 -- vault kv put secret/production/redis \
  password="RedisSecretPassword!" \
  host="redis.production.svc.cluster.local"

# 建立 Policy（最小權限）
kubectl exec -n vault vault-0 -- vault policy write production-app - <<EOF
# 允許讀取 production 路徑的機密
path "secret/data/production/*" {
  capabilities = ["read", "list"]
}

# 不允許寫入或刪除
path "secret/metadata/production/*" {
  capabilities = ["list", "read"]
}
EOF

# 建立 K8s Auth Role
kubectl exec -n vault vault-0 -- vault write auth/kubernetes/role/production-app \
  bound_service_account_names=myapp-sa \
  bound_service_account_namespaces=production \
  policies=production-app \
  ttl=1h
```

### 7.5 Pod 自動注入 Vault Secret

```yaml
---
# Pod 透過 Vault Agent Injector 自動取得機密
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
      annotations:
        # 啟用 Vault Agent Injector
        vault.hashicorp.com/agent-inject: "true"
        # Vault 角色
        vault.hashicorp.com/role: "production-app"
        # 注入資料庫機密
        vault.hashicorp.com/agent-inject-secret-database.env: "secret/data/production/database"
        # 自訂輸出格式（env 格式，方便應用程式讀取）
        vault.hashicorp.com/agent-inject-template-database.env: |
          {{- with secret "secret/data/production/database" -}}
          export DB_HOST="{{ .Data.data.host }}"
          export DB_PORT="{{ .Data.data.port }}"
          export DB_USER="{{ .Data.data.username }}"
          export DB_PASS="{{ .Data.data.password }}"
          {{- end }}
        # 注入 Redis 機密
        vault.hashicorp.com/agent-inject-secret-redis.env: "secret/data/production/redis"
        vault.hashicorp.com/agent-inject-template-redis.env: |
          {{- with secret "secret/data/production/redis" -}}
          export REDIS_HOST="{{ .Data.data.host }}"
          export REDIS_PASS="{{ .Data.data.password }}"
          {{- end }}
        # Agent 資源限制
        vault.hashicorp.com/agent-requests-cpu: "50m"
        vault.hashicorp.com/agent-requests-mem: "64Mi"
        vault.hashicorp.com/agent-limits-cpu: "100m"
        vault.hashicorp.com/agent-limits-mem: "128Mi"
    spec:
      serviceAccountName: myapp-sa
      containers:
      - name: myapp
        image: myapp:v1.0.0
        command: ["/bin/sh", "-c"]
        args:
        # 先 source 機密，再啟動應用程式
        - |
          source /vault/secrets/database.env
          source /vault/secrets/redis.env
          exec /app/myapp
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
          capabilities:
            drop: [ALL]
```

### 7.6 ArgoCD 整合 Vault（argocd-vault-plugin）

```bash
# 安裝 argocd-vault-plugin
# 在 ArgoCD 的 argocd-repo-server 中加入 plugin

# 方法：使用 init container 安裝 plugin
```

```yaml
# argocd-repo-server deployment patch
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-repo-server
  namespace: argocd
spec:
  template:
    spec:
      volumes:
      - name: custom-tools
        emptyDir: {}
      initContainers:
      - name: download-tools
        image: alpine:3.18
        command: [sh, -c]
        args:
        - |
          wget -O /custom-tools/argocd-vault-plugin \
            https://github.com/argoproj-labs/argocd-vault-plugin/releases/download/v1.17.0/argocd-vault-plugin_1.17.0_linux_amd64
          chmod +x /custom-tools/argocd-vault-plugin
        volumeMounts:
        - mountPath: /custom-tools
          name: custom-tools
      containers:
      - name: argocd-repo-server
        volumeMounts:
        - mountPath: /usr/local/bin/argocd-vault-plugin
          name: custom-tools
          subPath: argocd-vault-plugin
        env:
        - name: VAULT_ADDR
          value: "https://vault.vault.svc.cluster.local:8200"
        - name: VAULT_AUTH_TYPE
          value: "k8s"
        - name: VAULT_ROLE
          value: "argocd"
        - name: VAULT_SKIP_VERIFY
          value: "false"
```

```yaml
# ArgoCD Application 使用 Vault Plugin
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  source:
    repoURL: https://gitlab.company.com/myteam/myapp.git
    targetRevision: HEAD
    path: k8s/
    plugin:
      name: argocd-vault-plugin
      env:
      - name: AVP_TYPE
        value: vault
  destination:
    server: https://kubernetes.default.svc
    namespace: production
```

```yaml
# K8s 資源檔案中使用 placeholder（會被 argocd-vault-plugin 替換）
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secret
type: Opaque
stringData:
  database-url: <path:secret/data/production/database#host>
  database-password: <path:secret/data/production/database#password>
```

---

## 8. 稽核日誌

### 8.1 kube-apiserver 稽核設定

```bash
# 在每台 master 節點確保以下目錄存在
mkdir -p /var/log/kubernetes
mkdir -p /etc/kubernetes

# 複製 audit-policy.yaml 到 master 節點
cp audit-policy.yaml /etc/kubernetes/audit-policy.yaml
```

### 8.2 完整 audit-policy.yaml

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy

# 不記錄 RequestReceived 階段（減少重複）
omitStages:
- RequestReceived

rules:
# Level 說明：
# None    = 不記錄
# Metadata = 只記錄 metadata（無 request/response body）
# Request  = 記錄 metadata 和 request body
# RequestResponse = 記錄所有內容

# Secret 操作記錄完整內容（敏感操作）
- level: RequestResponse
  verbs: ["create", "update", "delete", "patch"]
  resources:
  - group: ""
    resources: ["secrets"]
  - group: ""
    resources: ["configmaps"]

# RBAC 變更記錄完整內容
- level: RequestResponse
  verbs: ["create", "update", "delete", "patch"]
  resources:
  - group: "rbac.authorization.k8s.io"
    resources: ["roles", "clusterroles", "rolebindings", "clusterrolebindings"]

# 認證失敗記錄
- level: Metadata
  omitStages:
  - RequestReceived
  userGroups: ["system:unauthenticated"]

# 系統 ServiceAccount 只記錄 Metadata（量大）
- level: None
  users:
  - system:kube-proxy
  - system:kube-controller-manager
  - system:kube-scheduler
  - system:node

# 節點 kubelet 操作
- level: None
  userGroups: ["system:nodes"]
  verbs: ["get"]
  resources:
  - group: ""
    resources: ["nodes", "pods"]

# 健康檢查不記錄
- level: None
  nonResourceURLs:
  - /healthz
  - /readyz
  - /livez
  - /metrics

# Helm/ArgoCD 操作記錄
- level: Request
  users: ["argocd-application-controller"]
  verbs: ["create", "update", "delete", "patch"]

# 刪除操作記錄完整內容（高風險操作）
- level: RequestResponse
  verbs: ["delete", "deletecollection"]

# Exec 和 Port-forward（高風險）
- level: RequestResponse
  resources:
  - group: ""
    resources: ["pods/exec", "pods/portforward", "pods/proxy"]

# 其他所有操作記錄 Metadata
- level: Metadata
```

**kube-apiserver 加入稽核設定**

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --audit-log-path=/var/log/kubernetes/audit.log
    - --audit-log-maxage=30          # 保留 30 天
    - --audit-log-maxbackup=10       # 最多 10 個備份
    - --audit-log-maxsize=100        # 每個 log 檔最大 100MB
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    # 也可以傳送到 webhook
    - --audit-webhook-config-file=/etc/kubernetes/audit-webhook.yaml
    - --audit-webhook-batch-max-wait=5s
    volumeMounts:
    - mountPath: /etc/kubernetes/audit-policy.yaml
      name: audit-policy
      readOnly: true
    - mountPath: /var/log/kubernetes
      name: audit-log
  volumes:
  - hostPath:
      path: /etc/kubernetes/audit-policy.yaml
      type: File
    name: audit-policy
  - hostPath:
      path: /var/log/kubernetes
      type: DirectoryOrCreate
    name: audit-log
```

### 8.3 稽核日誌分析

```bash
# 查詢特定使用者的操作
cat /var/log/kubernetes/audit.log | \
  jq 'select(.user.username == "admin@company.com")' | \
  jq '{time: .requestReceivedTimestamp, verb: .verb, resource: .objectRef.resource, name: .objectRef.name}'

# 查詢所有刪除操作
cat /var/log/kubernetes/audit.log | \
  jq 'select(.verb == "delete")' | \
  jq '{time: .requestReceivedTimestamp, user: .user.username, resource: .objectRef.resource, name: .objectRef.name, ns: .objectRef.namespace}'

# 查詢認證失敗
cat /var/log/kubernetes/audit.log | \
  jq 'select(.responseStatus.code == 401 or .responseStatus.code == 403)' | \
  jq '{time: .requestReceivedTimestamp, user: .user.username, reason: .responseStatus.reason}'

# 查詢 exec 操作
cat /var/log/kubernetes/audit.log | \
  jq 'select(.objectRef.subresource == "exec")' | \
  jq '{time: .requestReceivedTimestamp, user: .user.username, pod: .objectRef.name, ns: .objectRef.namespace}'
```

---

## 附錄：資安檢查清單

```
□ RBAC
  □ 無任何使用者/SA 擁有 cluster-admin（除 emergency 帳號）
  □ 所有 ServiceAccount 設定 automountServiceAccountToken: false
  □ 定期（每季）審查 RBAC 綁定關係

□ Network Policy
  □ 所有非 system namespace 都有 default-deny policy
  □ 僅開放必要的服務間通訊

□ Pod Security
  □ 所有 production namespace 設定 restricted PSS
  □ 所有容器設定 readOnlyRootFilesystem: true
  □ 所有容器移除所有 capabilities

□ 映像安全
  □ 所有映像都經過 Trivy 掃描（CI/CD gate）
  □ 無 CRITICAL 漏洞的映像才能部署到 production
  □ 使用 image digest 而非 tag（防止 tag 被覆蓋）

□ 機密管理
  □ 無機密存放在 Git repository
  □ 所有機密透過 Vault 管理
  □ K8s Secret 的 etcd 資料啟用加密

□ 監控告警
  □ Falco 正常運行並有告警通道
  □ Audit log 有收集與保存
  □ RBAC 違規事件有告警
```
