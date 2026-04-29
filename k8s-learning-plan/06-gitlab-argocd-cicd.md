# 06 - GitLab + ArgoCD CI/CD 完整指南

> 環境：GitLab（self-hosted VM）+ GitLab Runner（K8s）+ ArgoCD（K8s）
> K8s 叢集：3 master + 3 worker，Ubuntu 24.04
> CI/CD：GitLab CE self-hosted + ArgoCD on K8s

---

## 1. 架構說明

### 1.1 整體架構圖（ASCII）

```
開發者工作流程
─────────────────────────────────────────────────────────────────

開發者
  │
  │ git push
  ▼
┌─────────────────────────────────┐
│      GitLab (self-hosted VM)     │
│                                   │
│  ┌─────────────┐                 │
│  │  App Repo   │  ── Webhook ──► ArgoCD
│  │  (source)   │                 │
│  └─────────────┘                 │
│                                   │
│  ┌─────────────┐                 │
│  │ GitOps Repo │ ◄── CI 更新 ── │
│  │  (config)   │                 │
│  └─────────────┘                 │
│                                   │
│  ┌─────────────┐                 │
│  │  Container  │                 │
│  │  Registry   │                 │
│  └─────────────┘                 │
└─────────────────────────────────┘
          │
          │ CI Pipeline 觸發
          ▼
┌─────────────────────────────────┐
│       Kubernetes 叢集            │
│                                   │
│  ┌─────────────────────────┐    │
│  │    GitLab Runner (K8s)   │    │
│  │  ┌──────────────────┐   │    │
│  │  │  lint/test/build  │   │    │
│  │  │  scan/push/deploy │   │    │
│  │  └──────────────────┘   │    │
│  └─────────────────────────┘    │
│                                   │
│  ┌─────────────────────────┐    │
│  │       ArgoCD             │    │
│  │  ┌──────────────────┐   │    │
│  │  │  App Controller   │   │    │
│  │  │  Repo Server      │   │    │
│  │  │  API Server       │   │    │
│  │  └──────────────────┘   │    │
│  └─────────────────────────┘    │
│           │                       │
│           │ 部署/同步              │
│           ▼                       │
│  ┌─────────────────────────┐    │
│  │   應用程式 Namespace     │    │
│  │   (production/staging)   │    │
│  └─────────────────────────┘    │
└─────────────────────────────────┘
```

### 1.2 GitOps 流程說明

GitOps 的核心原則：**Git 是唯一的事實來源（Single Source of Truth）**

```
完整 GitOps 流程：

1. 開發者 push code 到 App Repo
         │
         ▼
2. GitLab CI Pipeline 自動觸發
   ├── lint：程式碼格式檢查
   ├── test：單元測試
   ├── build：Docker image 建置
   ├── scan：Trivy 安全掃描
   └── push：推送 image 到 Registry
         │
         ▼
3. CI Pipeline 更新 GitOps Config Repo
   └── 修改 Deployment 的 image tag
         │
         ▼
4. ArgoCD 偵測到 GitOps Repo 變更（Webhook/Polling）
         │
         ▼
5. ArgoCD 比對叢集現有狀態 vs Git 期望狀態
         │
         ▼
6. ArgoCD 自動（或手動確認後）同步至 K8s
         │
         ▼
7. K8s 部署新版應用程式
         │
         ▼
8. 健康狀態回報到 ArgoCD Dashboard
```

---

## 2. GitLab 部署（在獨立 VM 上）

### 2.1 環境準備

```bash
# 在 GitLab VM 執行（建議至少 4 CPU、8GB RAM、50GB disk）
# OS：Ubuntu 24.04

# 更新系統
apt update && apt upgrade -y

# 安裝必要套件
apt install -y curl openssh-server ca-certificates tzdata perl postfix

# 設定 postfix（選擇 Internet Site，設定郵件域名）
# 或使用 Null mailer（不需發送郵件的環境）

# 設定 hostname（替換為你的實際域名）
hostnamectl set-hostname gitlab.example.com

# 開放防火牆
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 22/tcp
ufw reload
```

### 2.2 安裝 GitLab CE

```bash
# 添加 GitLab 套件來源
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | bash

# 安裝 GitLab CE（指定版本，建議用 LTS 版本）
EXTERNAL_URL="http://gitlab.example.com" apt install -y gitlab-ce

# 若需要 HTTPS
EXTERNAL_URL="https://gitlab.example.com" apt install -y gitlab-ce

# 等待安裝完成後，取得初始 root 密碼
cat /etc/gitlab/initial_root_password
# 密碼有效期 24 小時，請立即登入修改

# 啟動 GitLab（安裝時已自動 reconfigure）
gitlab-ctl status
```

### 2.3 重要的 gitlab.rb 設定

```ruby
# /etc/gitlab/gitlab.rb 重要設定項目

# ===== 基本設定 =====
external_url 'https://gitlab.example.com'

# ===== Email 設定（使用外部 SMTP）=====
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.gmail.com"
gitlab_rails['smtp_port'] = 587
gitlab_rails['smtp_user_name'] = "your-email@gmail.com"
gitlab_rails['smtp_password'] = "your-app-password"
gitlab_rails['smtp_domain'] = "gmail.com"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['gitlab_email_from'] = 'gitlab@example.com'

# ===== Container Registry 設定 =====
registry_external_url 'https://registry.example.com'
gitlab_rails['registry_enabled'] = true

# ===== 效能調整 =====
# 減少 worker 數量（資源不足時）
puma['worker_processes'] = 2
puma['min_threads'] = 1
puma['max_threads'] = 4

# 調整 Sidekiq
sidekiq['concurrency'] = 10

# PostgreSQL 調整
postgresql['shared_buffers'] = "256MB"
postgresql['max_connections'] = 100

# ===== Nginx SSL 設定（讓 GitLab 管理憑證）=====
nginx['enable'] = true
nginx['redirect_http_to_https'] = true
nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab.example.com.crt"
nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab.example.com.key"

# ===== Let's Encrypt 自動憑證（若有公開域名）=====
letsencrypt['enable'] = true
letsencrypt['contact_emails'] = ['admin@example.com']
letsencrypt['auto_renew'] = true
letsencrypt['auto_renew_hour'] = 12
letsencrypt['auto_renew_minute'] = 30
letsencrypt['auto_renew_day_of_month'] = "*/7"

# ===== 備份設定 =====
gitlab_rails['backup_path'] = "/var/opt/gitlab/backups"
gitlab_rails['backup_keep_time'] = 604800   # 保留 7 天
```

```bash
# 修改 gitlab.rb 後套用設定
gitlab-ctl reconfigure

# 確認 GitLab 服務狀態
gitlab-ctl status

# 測試 GitLab 設定
gitlab-rake gitlab:check SANITIZE=true
```

### 2.4 SSL 設定（自簽憑證）

```bash
# 若無法使用 Let's Encrypt，使用自簽憑證

# 建立 SSL 目錄
mkdir -p /etc/gitlab/ssl

# 產生自簽憑證（有效期 3650 天）
openssl req -x509 -nodes -days 3650 -newkey rsa:4096 \
  -keyout /etc/gitlab/ssl/gitlab.example.com.key \
  -out /etc/gitlab/ssl/gitlab.example.com.crt \
  -subj "/C=TW/ST=Taiwan/L=Taipei/O=MyOrg/CN=gitlab.example.com" \
  -addext "subjectAltName=DNS:gitlab.example.com,IP:192.168.100.x"

chmod 600 /etc/gitlab/ssl/gitlab.example.com.key

# 套用設定
gitlab-ctl reconfigure
```

### 2.5 建立 K8s GitLab Runner

#### 2.5.1 取得 Runner Registration Token

```
1. 登入 GitLab Web UI
2. 進入 Admin Area -> CI/CD -> Runners
3. 點擊 "Register an instance runner"
4. 複製 registration token（格式：glrt-xxxxxxxxxxxx）

或取得 Group/Project 層級的 token：
- Group：Groups -> [group] -> CI/CD -> Runners
- Project：Project -> Settings -> CI/CD -> Runners
```

#### 2.5.2 GitLab Runner Helm values 設定

```yaml
# gitlab-runner-values.yaml

# GitLab 位址
gitlabUrl: "https://gitlab.example.com"

# Runner Registration Token（從 GitLab UI 取得）
runnerToken: "glrt-xxxxxxxxxxxxxxxxxxxx"

# Runner 設定
concurrent: 10              # 最多同時執行幾個 Job

# Runner Pod 設定
rbac:
  create: true
  clusterWideAccess: false  # 設 true 讓 runner 可存取所有 namespace

runners:
  config: |
    [[runners]]
      name = "k8s-runner"
      url = "https://gitlab.example.com"
      executor = "kubernetes"

      [runners.kubernetes]
        namespace = "gitlab-runner"
        image = "ubuntu:22.04"
        privileged = true         # Docker-in-Docker 需要

        # 資源限制
        cpu_limit = "2"
        memory_limit = "4Gi"
        cpu_limit_overwrite_max_allowed = "4"
        memory_limit_overwrite_max_allowed = "8Gi"

        # Service Account
        service_account = "gitlab-runner"

        # 拉取 image 的 policy
        pull_policy = "always"

        [runners.kubernetes.node_selector]
          "kubernetes.io/os" = "linux"

        # 掛載 Docker socket（DinD 方式）
        [[runners.kubernetes.volumes.host_path]]
          name = "docker-sock"
          mount_path = "/var/run/docker.sock"
          host_path = "/var/run/docker.sock"

        # 快取設定（使用 PVC）
        [[runners.kubernetes.volumes.pvc]]
          name = "runner-cache"
          mount_path = "/cache"
          claim_name = "gitlab-runner-cache-pvc"

  cache:
    secretName: "gitlab-runner-cache-secret"

# Runner Pod 資源限制
resources:
  limits:
    memory: 256Mi
    cpu: 200m
  requests:
    memory: 128Mi
    cpu: 100m

# 使用 CEPH StorageClass 作為 cache
persistence:
  enabled: true
  storageClass: "ceph-rbd"
  size: 20Gi
```

#### 2.5.3 安裝 GitLab Runner

```bash
# 在 K8s master 執行

# 添加 GitLab Helm repo
helm repo add gitlab https://charts.gitlab.io/
helm repo update

# 建立 namespace
kubectl create namespace gitlab-runner

# 建立 runner cache PVC
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-runner-cache-pvc
  namespace: gitlab-runner
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: ceph-rbd
EOF

# 安裝 GitLab Runner
helm install gitlab-runner gitlab/gitlab-runner \
  --namespace gitlab-runner \
  --version 0.61.0 \
  -f gitlab-runner-values.yaml

# 確認 Runner 狀態
kubectl -n gitlab-runner get pods
kubectl -n gitlab-runner logs -l app=gitlab-runner

# 在 GitLab UI 確認 Runner 已註冊
# Admin Area -> CI/CD -> Runners -> 應看到 k8s-runner 顯示為綠色
```

---

## 3. GitLab CI Pipeline 設計

### 3.1 完整 .gitlab-ci.yml 範例

```yaml
# .gitlab-ci.yml
# 完整 CI Pipeline：lint -> test -> build -> scan -> push -> deploy

# ===== 全域設定 =====
default:
  image: ubuntu:22.04
  before_script:
    - apt-get update -qq && apt-get install -y -qq curl git

variables:
  # Container Registry 設定
  REGISTRY: registry.example.com
  IMAGE_NAME: $REGISTRY/$CI_PROJECT_PATH
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA

  # GitOps Repo 設定
  GITOPS_REPO: "https://oauth2:${GITOPS_TOKEN}@gitlab.example.com/myorg/gitops-config.git"

  # Docker 設定（DinD）
  DOCKER_HOST: tcp://docker:2376
  DOCKER_TLS_CERTDIR: "/certs"
  DOCKER_TLS_VERIFY: 1
  DOCKER_CERT_PATH: "$DOCKER_TLS_CERTDIR/client"

# ===== Stages 定義 =====
stages:
  - lint
  - test
  - build
  - scan
  - push
  - deploy

# ===== Stage: lint =====
lint:python:
  stage: lint
  image: python:3.11-slim
  before_script:
    - pip install flake8 black isort pylint --quiet
  script:
    - echo ">>> 執行 flake8 語法檢查"
    - flake8 . --max-line-length=120 --exclude=.git,venv,__pycache__
    - echo ">>> 執行 black 格式檢查"
    - black --check --diff .
    - echo ">>> 執行 isort import 排序檢查"
    - isort --check-only --diff .
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_BRANCH =~ /^release\/.*/
  cache:
    key: pip-$CI_COMMIT_REF_SLUG
    paths:
      - .cache/pip/
  artifacts:
    when: on_failure
    expire_in: 1 week
    reports:
      # 若使用 pylint 可輸出 codeclimate 格式
      codequality: gl-code-quality-report.json

lint:dockerfile:
  stage: lint
  image: hadolint/hadolint:latest-debian
  script:
    - echo ">>> 執行 Dockerfile 最佳實踐檢查"
    - hadolint Dockerfile
  rules:
    - changes:
        - Dockerfile

# ===== Stage: test =====
test:unit:
  stage: test
  image: python:3.11-slim
  services:
    - name: redis:7-alpine
      alias: redis
    - name: postgres:15-alpine
      alias: postgres
      variables:
        POSTGRES_DB: testdb
        POSTGRES_USER: testuser
        POSTGRES_PASSWORD: testpass
  variables:
    DATABASE_URL: "postgresql://testuser:testpass@postgres:5432/testdb"
    REDIS_URL: "redis://redis:6379/0"
  before_script:
    - pip install -r requirements.txt -r requirements-dev.txt --quiet
  script:
    - echo ">>> 執行單元測試"
    - pytest tests/unit/ \
        --junitxml=junit-report.xml \
        --cov=app \
        --cov-report=xml:coverage.xml \
        --cov-report=term-missing \
        --cov-fail-under=70 \
        -v
  coverage: '/TOTAL.*\s+(\d+%)$/'
  artifacts:
    when: always
    expire_in: 1 week
    reports:
      junit: junit-report.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
  cache:
    key: pip-$CI_COMMIT_REF_SLUG
    paths:
      - .cache/pip/

test:integration:
  stage: test
  image: python:3.11-slim
  services:
    - name: postgres:15-alpine
      alias: postgres
      variables:
        POSTGRES_DB: testdb
        POSTGRES_USER: testuser
        POSTGRES_PASSWORD: testpass
  variables:
    DATABASE_URL: "postgresql://testuser:testpass@postgres:5432/testdb"
  before_script:
    - pip install -r requirements.txt -r requirements-dev.txt --quiet
  script:
    - echo ">>> 執行整合測試"
    - pytest tests/integration/ \
        --junitxml=integration-junit-report.xml \
        -v
  artifacts:
    when: always
    reports:
      junit: integration-junit-report.xml
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_BRANCH =~ /^release\/.*/

# ===== Stage: build =====
build:image:
  stage: build
  image: docker:24.0
  services:
    - name: docker:24.0-dind
      variables:
        DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - echo ">>> 建置 Docker image"
    - |
      docker build \
        --build-arg BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
        --build-arg GIT_COMMIT=$CI_COMMIT_SHORT_SHA \
        --build-arg GIT_BRANCH=$CI_COMMIT_REF_NAME \
        --cache-from $IMAGE_NAME:latest \
        --tag $IMAGE_NAME:$IMAGE_TAG \
        --tag $IMAGE_NAME:$CI_COMMIT_REF_SLUG \
        .
    - echo ">>> 儲存 image tar（供後續 stage 使用）"
    - docker save $IMAGE_NAME:$IMAGE_TAG > image.tar
  artifacts:
    paths:
      - image.tar
    expire_in: 2 hours
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_BRANCH =~ /^release\/.*/
    - if: $CI_COMMIT_TAG

# ===== Stage: scan =====
scan:trivy:
  stage: scan
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  before_script:
    - docker load -i image.tar
  script:
    - echo ">>> 執行 Trivy 容器安全掃描"
    - |
      trivy image \
        --format template \
        --template "@/contrib/gitlab.tpl" \
        --output trivy-report.json \
        --severity CRITICAL,HIGH \
        --exit-code 0 \
        $IMAGE_NAME:$IMAGE_TAG
    - echo ">>> 輸出掃描結果"
    - trivy image --severity CRITICAL,HIGH --exit-code 0 $IMAGE_NAME:$IMAGE_TAG
    - echo ">>> 若有 CRITICAL 漏洞則失敗"
    - trivy image --severity CRITICAL --exit-code 1 $IMAGE_NAME:$IMAGE_TAG
  artifacts:
    when: always
    expire_in: 1 week
    reports:
      container_scanning: trivy-report.json
  dependencies:
    - build:image
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_BRANCH =~ /^release\/.*/
    - if: $CI_COMMIT_TAG

scan:secret-detection:
  stage: scan
  image: trufflesecurity/trufflehog:latest
  script:
    - echo ">>> 掃描 secret 洩漏"
    - trufflehog git file://. --only-verified
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

# ===== Stage: push =====
push:registry:
  stage: push
  image: docker:24.0
  services:
    - docker:24.0-dind
  before_script:
    - docker load -i image.tar
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - echo ">>> 推送 image 到 GitLab Container Registry"
    - docker push $IMAGE_NAME:$IMAGE_TAG
    - docker push $IMAGE_NAME:$CI_COMMIT_REF_SLUG
    - |
      if [[ "$CI_COMMIT_BRANCH" == "$CI_DEFAULT_BRANCH" ]]; then
        docker tag $IMAGE_NAME:$IMAGE_TAG $IMAGE_NAME:latest
        docker push $IMAGE_NAME:latest
      fi
    - echo "IMAGE_TAG=$IMAGE_TAG" > build.env
  artifacts:
    reports:
      dotenv: build.env
  dependencies:
    - build:image
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_BRANCH =~ /^release\/.*/
    - if: $CI_COMMIT_TAG

# ===== Stage: deploy =====
deploy:update-gitops:
  stage: deploy
  image: alpine/git:latest
  variables:
    GIT_STRATEGY: none    # 不 clone 目前 repo，我們要操作 gitops repo
  before_script:
    - apk add --no-cache git curl
    - git config --global user.email "ci@example.com"
    - git config --global user.name "GitLab CI"
  script:
    - echo ">>> Clone GitOps Config Repo"
    - git clone $GITOPS_REPO /tmp/gitops
    - cd /tmp/gitops
    - echo ">>> 更新 image tag"
    - |
      # 更新 staging 環境的 image tag
      sed -i "s|image: ${REGISTRY}/${CI_PROJECT_PATH}:.*|image: ${REGISTRY}/${CI_PROJECT_PATH}:${IMAGE_TAG}|" \
        apps/myapp/overlays/staging/kustomization.yaml
    - git add -A
    - git diff --staged
    - git commit -m "ci: update myapp image to $IMAGE_TAG [skip ci]"
    - git push origin main
    - echo ">>> GitOps Repo 已更新，ArgoCD 將自動同步"
  environment:
    name: staging
    url: https://myapp-staging.example.com
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

deploy:trigger-argocd:
  stage: deploy
  image: argoproj/argocd:latest
  script:
    - echo ">>> 登入 ArgoCD"
    - argocd login $ARGOCD_SERVER \
        --username $ARGOCD_USER \
        --password $ARGOCD_PASS \
        --insecure
    - echo ">>> 觸發 ArgoCD Sync"
    - argocd app sync myapp-staging --async
    - echo ">>> 等待同步完成"
    - argocd app wait myapp-staging --health --timeout 300
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  needs:
    - deploy:update-gitops
```

### 3.2 Dockerfile 最佳實踐（Multi-stage Build）

```dockerfile
# Dockerfile
# Python 應用程式範例（Multi-stage Build）

# ===== Stage 1: 依賴安裝 =====
FROM python:3.11-slim AS builder

# 設定工作目錄
WORKDIR /build

# 只複製依賴檔案（提升 cache 效率）
COPY requirements.txt .

# 使用虛擬環境隔離依賴
RUN python -m venv /opt/venv && \
    /opt/venv/bin/pip install --upgrade pip --quiet && \
    /opt/venv/bin/pip install --no-cache-dir -r requirements.txt --quiet

# ===== Stage 2: 最終 image =====
FROM python:3.11-slim AS runtime

# 安全：以非 root 使用者執行
RUN groupadd -r appgroup && useradd -r -g appgroup -u 1001 appuser

# 設定工作目錄
WORKDIR /app

# 從 builder stage 複製虛擬環境
COPY --from=builder /opt/venv /opt/venv

# 複製應用程式碼
COPY --chown=appuser:appgroup . .

# 設定 PATH 使用虛擬環境
ENV PATH="/opt/venv/bin:$PATH"
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

# 設定 build metadata（由 CI 傳入）
ARG BUILD_DATE
ARG GIT_COMMIT
ARG GIT_BRANCH
LABEL org.opencontainers.image.created=$BUILD_DATE \
      org.opencontainers.image.revision=$GIT_COMMIT \
      org.opencontainers.image.source=$GIT_BRANCH

# 切換至非 root 使用者
USER appuser

# 暴露 Port
EXPOSE 8000

# 健康檢查
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

# 啟動指令
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "2", "--threads", "4", "app.main:app"]
```

### 3.3 GitLab Container Registry 設定

```bash
# 確認 Registry 已啟用
# gitlab.rb 設定（已在 2.3 節設定）

# 在 CI/CD 中使用 Registry 的環境變數（GitLab 自動注入）：
# CI_REGISTRY          = registry.example.com
# CI_REGISTRY_IMAGE    = registry.example.com/group/project
# CI_REGISTRY_USER     = gitlab-ci-token
# CI_REGISTRY_PASSWORD = <自動產生的 token>

# 手動登入 Registry 測試
docker login registry.example.com -u your-username -p your-token

# 查看 Project 的 Container Registry
# GitLab UI -> Project -> Packages & Registries -> Container Registry
```

---

## 4. ArgoCD 部署與設定

### 4.1 安裝 ArgoCD（Helm）

```bash
# 在 K8s master 執行

# 添加 ArgoCD Helm repo
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# 建立 namespace
kubectl create namespace argocd

# 建立 argocd-values.yaml
cat > argocd-values.yaml << 'EOF'
# ArgoCD Helm Values

global:
  image:
    tag: "v2.10.0"

# ===== ArgoCD Server =====
server:
  replicas: 2            # HA 模式

  # Service 設定
  service:
    type: ClusterIP

  # Ingress 設定（需要先安裝 Ingress Controller）
  ingress:
    enabled: true
    ingressClassName: nginx
    hostname: argocd.example.com
    tls: true
    annotations:
      nginx.ingress.kubernetes.io/ssl-passthrough: "true"
      nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"

  # 額外啟動參數
  extraArgs:
    - --insecure       # 若前面有 Ingress 處理 TLS，可加此參數

  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 100m
      memory: 128Mi

# ===== Application Controller =====
controller:
  replicas: 1

  resources:
    limits:
      cpu: 1000m
      memory: 1Gi
    requests:
      cpu: 250m
      memory: 256Mi

# ===== Repo Server =====
repoServer:
  replicas: 2

  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 100m
      memory: 256Mi

# ===== Redis =====
redis:
  enabled: true
  resources:
    limits:
      cpu: 200m
      memory: 256Mi

# ===== ApplicationSet Controller =====
applicationSet:
  replicas: 2
  resources:
    limits:
      cpu: 200m
      memory: 256Mi

# ===== 設定 =====
configs:
  # RBAC 設定
  rbac:
    policy.default: role:readonly
    policy.csv: |
      g, myorg:devops, role:admin
      g, myorg:developers, role:developer

  # CM 設定（argocd-cm）
  cm:
    # 連接的 GitLab（加入 OIDC 或 local users）
    url: https://argocd.example.com
    # 啟用 status badge
    statusbadge.enabled: "true"
    # Webhook 設定（啟用 GitLab webhook）
    admin.enabled: "true"
    # 應用程式健康檢查設定
    resource.customizations.health.apps_Deployment: |
      hs = {}
      hs.status = "Progressing"
      hs.message = ""
      if obj.status ~= nil then
        if obj.status.readyReplicas == obj.status.replicas then
          hs.status = "Healthy"
        end
      end
      return hs

  # Secret 設定（argocd-secret）
  secret:
    # 初始 admin 密碼（bcrypt hash）
    # 使用以下指令產生：htpasswd -nbBC 12 "" your-password | tr -d ':\n'
    argocdServerAdminPassword: "$2a$12$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
    argocdServerAdminPasswordMtime: "2024-01-01T00:00:00Z"

# ===== 通知設定（Notifications）=====
notifications:
  enabled: true
  secret:
    create: true
  metrics:
    enabled: false
EOF

# 安裝 ArgoCD
helm install argocd argo/argo-cd \
  --namespace argocd \
  --version 6.7.0 \
  -f argocd-values.yaml

# 確認安裝狀態
kubectl -n argocd get pods
watch kubectl -n argocd get pods
```

### 4.2 取得初始密碼並修改

```bash
# 取得初始 admin 密碼
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
echo ""

# Port-forward 到本地（若尚未設定 Ingress）
kubectl -n argocd port-forward svc/argocd-server 8080:443 &

# 或直接透過 Ingress 存取
# https://argocd.example.com
```

### 4.3 安裝 ArgoCD CLI 工具

```bash
# 安裝 argocd CLI（在 master 節點或本機）

# Linux
curl -sSL -o /usr/local/bin/argocd \
  https://github.com/argoproj/argo-cd/releases/download/v2.10.0/argocd-linux-amd64
chmod +x /usr/local/bin/argocd

# 確認版本
argocd version --client

# 登入 ArgoCD
argocd login argocd.example.com \
  --username admin \
  --password "$(kubectl -n argocd get secret argocd-initial-admin-secret \
    -o jsonpath='{.data.password}' | base64 -d)" \
  --insecure

# 修改密碼
argocd account update-password \
  --current-password "$(kubectl -n argocd get secret argocd-initial-admin-secret \
    -o jsonpath='{.data.password}' | base64 -d)" \
  --new-password "YourNewSecurePassword123!"

# 刪除初始密碼 Secret
kubectl -n argocd delete secret argocd-initial-admin-secret

# 確認登入成功
argocd account list
argocd cluster list
```

### 4.4 連接 GitLab Repo

```bash
# 方法一：使用 HTTPS（Personal Access Token）
argocd repo add https://gitlab.example.com/myorg/gitops-config.git \
  --username argocd-bot \
  --password glpat-xxxxxxxxxxxxxxxxxxxx \
  --insecure-skip-server-verification

# 方法二：使用 SSH key
# 產生 SSH key
ssh-keygen -t ed25519 -C "argocd@example.com" -f /tmp/argocd-deploy-key -N ""

# 查看公鑰（複製到 GitLab Project -> Settings -> Repository -> Deploy keys）
cat /tmp/argocd-deploy-key.pub

# 添加 SSH repo
argocd repo add git@gitlab.example.com:myorg/gitops-config.git \
  --ssh-private-key-path /tmp/argocd-deploy-key \
  --insecure-ignore-host-key

# 確認 repo 連接
argocd repo list
```

### 4.5 建立 ArgoCD Application（YAML 範例）

```yaml
# argocd-app-myapp-staging.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-staging
  namespace: argocd
  # 設定 finalizer 確保刪除 App 時同步刪除 K8s 資源
  finalizers:
    - resources-finalizer.argocd.argoproj.io
  labels:
    app.kubernetes.io/name: myapp
    environment: staging
spec:
  # ArgoCD Project（用於 RBAC）
  project: default

  # 來源：GitOps Config Repo
  source:
    repoURL: https://gitlab.example.com/myorg/gitops-config.git
    targetRevision: main
    path: apps/myapp/overlays/staging

  # 目標：K8s 叢集及 Namespace
  destination:
    server: https://kubernetes.default.svc  # 同叢集
    namespace: myapp-staging

  # 同步策略
  syncPolicy:
    # 自動同步（偵測到 Git 差異後自動 apply）
    automated:
      prune: true      # 刪除 Git 中不存在的 K8s 資源
      selfHeal: true   # 若 K8s 資源被手動修改，自動恢復

    syncOptions:
      - CreateNamespace=true    # 自動建立 namespace
      - PrunePropagationPolicy=foreground
      - PruneLast=true

    # 重試設定
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  # 健康狀態評估忽略特定差異
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas    # 若使用 HPA，忽略 replicas 差異
```

```bash
# 套用 Application
kubectl apply -f argocd-app-myapp-staging.yaml

# 確認 Application 狀態
argocd app list
argocd app get myapp-staging

# 手動同步
argocd app sync myapp-staging

# 查看同步歷史
argocd app history myapp-staging

# 回滾到特定版本
argocd app rollback myapp-staging <revision-id>
```

### 4.6 App of Apps 模式說明

App of Apps 是一種用 ArgoCD 管理多個 Application 的模式：一個「根 Application」負責管理其他 Application 的部署。

```
gitops-config/
└── argocd/
    ├── root-app.yaml          ← 根 Application（手動套用一次）
    └── applications/
        ├── myapp-staging.yaml
        ├── myapp-production.yaml
        ├── monitoring-stack.yaml
        └── ingress-nginx.yaml
```

```yaml
# argocd/root-app.yaml（根 Application）
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://gitlab.example.com/myorg/gitops-config.git
    targetRevision: main
    path: argocd/applications    # 指向所有 Application YAML 的目錄
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd            # Application CR 部署在 argocd namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
# 只需手動套用一次根 Application
kubectl apply -f argocd/root-app.yaml

# 之後新增應用程式只需在 argocd/applications/ 目錄下添加 YAML
# ArgoCD 會自動偵測並建立新的 Application
```

### 4.7 ArgoCD 與 GitLab 整合（Webhook）

```bash
# 設定 GitLab Webhook，讓 Git push 後立即通知 ArgoCD

# 1. 取得 ArgoCD Webhook Secret
kubectl -n argocd get secret argocd-secret \
  -o jsonpath='{.data.webhook\.gitlab\.secret}' | base64 -d

# 若尚未設定，手動建立 webhook secret
WEBHOOK_SECRET=$(openssl rand -hex 32)
kubectl -n argocd patch secret argocd-secret \
  -p "{\"stringData\": {\"webhook.gitlab.secret\": \"$WEBHOOK_SECRET\"}}"
echo "Webhook Secret: $WEBHOOK_SECRET"

# 2. 在 GitLab 設定 Webhook
# 路徑：GitLab -> [Group/Project] -> Settings -> Webhooks
# URL：https://argocd.example.com/api/webhook
# Secret Token：上面取得的 WEBHOOK_SECRET
# 勾選觸發事件：Push events、Tag push events
# 取消 SSL Verify（若使用自簽憑證）

# 3. 測試 Webhook
# 在 GitLab Webhook 設定頁面點擊 "Test" -> "Push events"
# 確認 ArgoCD log 有收到通知
kubectl -n argocd logs deployment/argocd-server | grep webhook
```

---

## 5. GitOps 完整工作流程

### 5.1 應用程式 Repo 結構

```
myapp/                          # 應用程式 Repo（source code）
├── .gitlab-ci.yml              # CI Pipeline 定義
├── Dockerfile                  # 容器 image
├── Makefile                    # 本地開發工具
├── requirements.txt
├── requirements-dev.txt
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── config.py
│   └── api/
│       ├── __init__.py
│       └── routes.py
├── tests/
│   ├── unit/
│   │   └── test_routes.py
│   └── integration/
│       └── test_api.py
└── k8s/                        # 基礎 K8s manifests（不含環境特定設定）
    ├── base/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── kustomization.yaml
    └── README.md
```

### 5.2 GitOps Config Repo 結構

```
gitops-config/                  # GitOps 設定 Repo（環境設定）
├── argocd/
│   ├── root-app.yaml           # App of Apps 根 Application
│   └── applications/
│       ├── myapp-staging.yaml
│       ├── myapp-production.yaml
│       └── infra-apps.yaml
├── apps/
│   └── myapp/
│       ├── base/               # 共用基礎設定
│       │   ├── kustomization.yaml
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   ├── hpa.yaml
│       │   └── pdb.yaml
│       └── overlays/
│           ├── staging/
│           │   ├── kustomization.yaml    # ← CI 更新此檔案的 image tag
│           │   ├── replica-patch.yaml
│           │   └── configmap-patch.yaml
│           └── production/
│               ├── kustomization.yaml
│               ├── replica-patch.yaml
│               └── configmap-patch.yaml
└── infra/
    ├── ingress-nginx/
    │   ├── namespace.yaml
    │   └── values.yaml
    └── cert-manager/
        ├── namespace.yaml
        └── values.yaml
```

#### kustomization.yaml 範例

```yaml
# apps/myapp/overlays/staging/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: myapp-staging

bases:
  - ../../base

# CI Pipeline 會更新此處的 image tag
images:
  - name: registry.example.com/myorg/myapp
    newTag: "a1b2c3d4"    # ← 由 CI 自動更新此值

patchesStrategicMerge:
  - replica-patch.yaml
  - configmap-patch.yaml

commonLabels:
  environment: staging
  managed-by: argocd

commonAnnotations:
  deployment.kubernetes.io/revision: "1"
```

### 5.3 從 push code 到自動部署的完整流程

```
完整流程時序圖：

開發者                GitLab             GitLab CI          GitOps Repo        ArgoCD             K8s
  │                     │                   │                   │                 │                 │
  │── git push ────────►│                   │                   │                 │                 │
  │                     │── 觸發 Pipeline ─►│                   │                 │                 │
  │                     │                   │── lint stage ──── │                 │                 │
  │                     │                   │── test stage ──── │                 │                 │
  │                     │                   │── build stage ─── │                 │                 │
  │                     │                   │── scan stage ──── │                 │                 │
  │                     │                   │── push stage ──── │                 │                 │
  │                     │                   │   (image 推到 Registry)             │                 │
  │                     │                   │── deploy stage ──►│                 │                 │
  │                     │                   │   (更新 image tag) │                 │                 │
  │                     │                   │                   │── Webhook ─────►│                 │
  │                     │                   │                   │                 │── 比對差異 ─────►│
  │                     │                   │                   │                 │◄── 差異結果 ─────│
  │                     │                   │                   │                 │── Apply ────────►│
  │                     │                   │                   │                 │◄── 部署完成 ─────│
  │◄── 通知（可選）──────│◄── 通知 ──────────│                   │                 │                 │
  │                     │                   │                   │                 │                 │

時間估計：
- lint + test：約 2-5 分鐘
- build + scan：約 3-8 分鐘
- push + deploy：約 1-2 分鐘
- ArgoCD 同步：約 30 秒-2 分鐘
總計：約 7-17 分鐘（從 push 到部署完成）
```

### 5.4 Image Tag 更新策略

#### 策略一：CI 直接修改 GitOps Repo（本指南採用）

```bash
# .gitlab-ci.yml deploy stage 中的腳本
git clone $GITOPS_REPO /tmp/gitops
cd /tmp/gitops
sed -i "s|newTag: .*|newTag: \"${IMAGE_TAG}\"|" \
  apps/myapp/overlays/staging/kustomization.yaml
git commit -am "ci: update myapp to $IMAGE_TAG"
git push origin main
```

#### 策略二：ArgoCD Image Updater（自動偵測 Registry）

ArgoCD Image Updater 是一個附加元件，可以自動掃描 Container Registry 並更新 Git 中的 image tag。

```bash
# 安裝 ArgoCD Image Updater
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml

# 確認安裝
kubectl -n argocd get pods -l app.kubernetes.io/name=argocd-image-updater

# 設定 Registry 存取 Secret
kubectl create secret docker-registry argocd-image-updater-secret \
  --docker-server=registry.example.com \
  --docker-username=argocd-bot \
  --docker-password=glpat-xxxx \
  --namespace argocd
```

```yaml
# 在 Application CR 添加 Image Updater 的 annotation
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-staging
  namespace: argocd
  annotations:
    # 設定要追蹤的 image
    argocd-image-updater.argoproj.io/image-list: myapp=registry.example.com/myorg/myapp
    # 更新策略：semver（語義版本）或 latest（最新 tag）
    argocd-image-updater.argoproj.io/myapp.update-strategy: latest
    # 只追蹤符合 pattern 的 tag（避免追蹤 latest tag）
    argocd-image-updater.argoproj.io/myapp.allow-tags: regexp:^[0-9a-f]{8}$
    # 回寫方式（git = 更新 Git，argocd = 只更新 ArgoCD 記憶體）
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
spec:
  # ... 其他設定
```

---

## 6. 環境變數與 Secret 管理

### 6.1 GitLab CI/CD Variables 設定

```
在 GitLab UI 設定以下 CI/CD Variables：
路徑：Project/Group -> Settings -> CI/CD -> Variables

必要變數：
┌────────────────────────┬─────────────────────────────┬─────────┐
│ Variable               │ 說明                         │ Masked  │
├────────────────────────┼─────────────────────────────┼─────────┤
│ CI_REGISTRY_USER       │ GitLab Registry 使用者       │ No      │
│ CI_REGISTRY_PASSWORD   │ GitLab Registry Token        │ Yes     │
│ GITOPS_TOKEN           │ GitOps Repo Access Token     │ Yes     │
│ ARGOCD_SERVER          │ ArgoCD Server URL            │ No      │
│ ARGOCD_USER            │ ArgoCD 使用者名稱             │ No      │
│ ARGOCD_PASS            │ ArgoCD 密碼                  │ Yes     │
│ TRIVY_TIMEOUT          │ Trivy 掃描逾時（預設 5m0s）   │ No      │
└────────────────────────┴─────────────────────────────┴─────────┘
```

### 6.2 K8s 應用程式 Secret 管理

```bash
# 不建議將 Secret 直接放入 GitOps Repo
# 建議使用以下方案：

# 方案一：Sealed Secrets（推薦）
# 安裝 kubeseal
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system

# 建立 Secret 並加密
kubectl create secret generic myapp-secret \
  --from-literal=DB_PASSWORD=mypassword \
  --dry-run=client -o yaml | \
  kubeseal --format yaml > sealed-secret.yaml

# 將 sealed-secret.yaml 提交到 GitOps Repo（安全，因為加密過）
git add sealed-secret.yaml && git commit -m "add myapp sealed secret"

# 方案二：External Secrets Operator（整合 HashiCorp Vault / AWS SM）
# 方案三：GitLab CI 直接注入到 K8s Secret（不存 Git）
```

---

## 附錄：常見問題排查

### GitLab Runner 無法啟動 Job

```bash
# 查看 runner pod log
kubectl -n gitlab-runner logs -l app=gitlab-runner

# 確認 runner 已在 GitLab 註冊
# GitLab UI -> Admin -> Runners

# 重新安裝 runner（更換 token）
helm upgrade gitlab-runner gitlab/gitlab-runner \
  --namespace gitlab-runner \
  --set runnerToken="glrt-new-token" \
  -f gitlab-runner-values.yaml
```

### ArgoCD App 同步失敗

```bash
# 查看 App 狀態詳情
argocd app get myapp-staging --show-operation

# 查看同步錯誤
kubectl -n argocd logs deployment/argocd-application-controller | grep ERROR

# 強制同步（重置到 Git 狀態）
argocd app sync myapp-staging --force

# 若有資源衝突，使用 replace
argocd app sync myapp-staging --replace
```

### GitLab Webhook 未觸發 ArgoCD

```bash
# 確認 Webhook 設定正確
# GitLab -> Settings -> Webhooks -> 查看最近的事件

# 確認 ArgoCD Server 可被 GitLab 訪問
curl -I https://argocd.example.com/api/webhook

# 查看 ArgoCD Server log
kubectl -n argocd logs deployment/argocd-server | grep -i webhook
```
