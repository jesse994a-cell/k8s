# Kubernetes HA 叢集部署實戰指南

> 環境：Ubuntu 24.04 | 3 master + 3 worker | HAProxy + Keepalived | CNI: Calico | CRI: containerd

---

## 目錄

1. [HA 架構設計](#1-ha-架構設計)
2. [環境準備（所有節點）](#2-環境準備所有節點)
3. [HAProxy 安裝與設定](#3-haproxy-安裝與設定)
4. [Keepalived 安裝與設定](#4-keepalived-安裝與設定)
5. [containerd 安裝與設定](#5-containerd-安裝與設定)
6. [kubeadm / kubelet / kubectl 安裝](#6-kubeadm--kubelet--kubectl-安裝)
7. [初始化第一個 Master 節點](#7-初始化第一個-master-節點)
8. [加入其他 Master 節點](#8-加入其他-master-節點)
9. [加入 Worker 節點](#9-加入-worker-節點)
10. [Calico CNI 安裝](#10-calico-cni-安裝)
11. [叢集驗證](#11-叢集驗證)
12. [常見問題排除](#12-常見問題排除)

---

## 1. HA 架構設計

### 1.1 整體 HA 架構圖

```
                          ┌──────────────────────────────────┐
                          │  使用者 / kubectl / CI/CD        │
                          └─────────────┬────────────────────┘
                                        │ HTTPS:6443
                                        ▼
                    ┌───────────────────────────────────────┐
                    │   Virtual IP: 192.168.100.10:6443     │
                    │   (由 Keepalived 管理 VIP 飄移)        │
                    └────────┬──────────┬────────┬──────────┘
                             │          │        │  HAProxy 負載均衡
                    ┌────────▼──┐  ┌────▼───┐  ┌─▼────────┐
                    │ master1   │  │master2 │  │ master3  │
                    │.100.11    │  │.100.12 │  │ .100.13  │
                    │           │  │        │  │          │
                    │ HAProxy   │  │HAProxy │  │ HAProxy  │
                    │ Keepalived│  │Keepaliv│  │ Keepaliv │
                    │ apiserver │  │apiservr│  │ apiservr │
                    │ scheduler │  │schedul │  │ schedul  │
                    │ ctrl-mgr  │  │ctrl-mgr│  │ ctrl-mgr │
                    │ etcd      │  │ etcd   │  │  etcd    │
                    └─────┬─────┘  └───┬────┘  └────┬─────┘
                          │            │             │
          ┌───────────────┼────────────┼─────────────┼───────────────┐
          │               │            │             │               │
   ┌──────▼──────┐  ┌─────▼──────┐  ┌─▼──────────┐
   │  worker1    │  │  worker2   │  │  worker3   │
   │ .100.21     │  │  .100.22   │  │  .100.23   │
   │             │  │            │  │            │
   │ kubelet     │  │  kubelet   │  │  kubelet   │
   │ kube-proxy  │  │ kube-proxy │  │ kube-proxy │
   │ containerd  │  │ containerd │  │ containerd │
   └─────────────┘  └────────────┘  └────────────┘
```

### 1.2 節點規劃

| 角色 | 主機名稱 | IP 位址 | vCPU | RAM | 磁碟 |
|------|----------|---------|------|-----|------|
| VIP (HA) | - | 192.168.100.10 | - | - | - |
| master1 | k8s-master1 | 192.168.100.11 | 4 | 8GB | 60GB |
| master2 | k8s-master2 | 192.168.100.12 | 4 | 8GB | 60GB |
| master3 | k8s-master3 | 192.168.100.13 | 4 | 8GB | 60GB |
| worker1 | k8s-worker1 | 192.168.100.21 | 4 | 16GB | 100GB |
| worker2 | k8s-worker2 | 192.168.100.22 | 4 | 16GB | 100GB |
| worker3 | k8s-worker3 | 192.168.100.23 | 4 | 16GB | 100GB |

### 1.3 HA 機制說明

- **Keepalived**：使用 VRRP 協定，在 master 節點之間管理 VIP（192.168.100.10）的飄移。當 MASTER 節點故障，BACKUP 節點自動接管 VIP。
- **HAProxy**：在每個 master 節點上運行，監聽 VIP:6443，將請求負載均衡到三個 kube-apiserver（:6443）。
- **stacked etcd**：etcd 與 control plane 共同部署在 master 節點，透過 Raft 演算法保證資料一致性。

---

## 2. 環境準備（所有節點）

以下步驟需在**所有 6 個節點**（3 master + 3 worker）上執行。

### 2.1 設定主機名稱

```bash
# 在 master1 執行
hostnamectl set-hostname k8s-master1

# 在 master2 執行
hostnamectl set-hostname k8s-master2

# 在 master3 執行
hostnamectl set-hostname k8s-master3

# 在 worker1 執行
hostnamectl set-hostname k8s-worker1

# 在 worker2 執行
hostnamectl set-hostname k8s-worker2

# 在 worker3 執行
hostnamectl set-hostname k8s-worker3
```

### 2.2 設定 /etc/hosts（所有節點）

```bash
cat >> /etc/hosts << 'EOF'
192.168.100.10  k8s-vip
192.168.100.11  k8s-master1
192.168.100.12  k8s-master2
192.168.100.13  k8s-master3
192.168.100.21  k8s-worker1
192.168.100.22  k8s-worker2
192.168.100.23  k8s-worker3
EOF
```

### 2.3 關閉 Swap

```bash
# 立即關閉 swap
swapoff -a

# 永久關閉（註解掉 /etc/fstab 中的 swap 行）
sed -i '/\sswap\s/s/^/#/' /etc/fstab

# 驗證
free -h | grep Swap
# 預期：Swap 的數值全為 0
```

### 2.4 載入必要的核心模組

```bash
# 載入模組
cat > /etc/modules-load.d/k8s.conf << 'EOF'
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# 驗證模組已載入
lsmod | grep -E "overlay|br_netfilter"
```

### 2.5 設定核心網路參數

```bash
cat > /etc/sysctl.d/k8s.conf << 'EOF'
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF

# 套用設定
sysctl --system

# 驗證
sysctl net.bridge.bridge-nf-call-iptables net.ipv4.ip_forward
```

### 2.6 安裝必要工具

```bash
apt-get update && apt-get install -y \
  apt-transport-https \
  ca-certificates \
  curl \
  gnupg \
  lsb-release \
  socat \
  conntrack \
  ipvsadm \
  ipset \
  jq \
  wget \
  vim
```

### 2.7 關閉防火牆（測試環境）或設定規則

```bash
# 測試環境：關閉 UFW
systemctl stop ufw
systemctl disable ufw

# 生產環境：開放必要 Port
# Master 節點需開放：
# ufw allow 6443/tcp     # kube-apiserver
# ufw allow 2379:2380/tcp # etcd
# ufw allow 10250/tcp    # kubelet API
# ufw allow 10257/tcp    # kube-controller-manager
# ufw allow 10259/tcp    # kube-scheduler
# Worker 節點需開放：
# ufw allow 10250/tcp    # kubelet API
# ufw allow 30000:32767/tcp  # NodePort Services
```

---

## 3. HAProxy 安裝與設定

HAProxy 安裝在**所有 3 台 master 節點**上，監聽 VIP 的 6443 port，將流量分發到各 apiserver。

### 3.1 安裝 HAProxy

```bash
# 在所有 master 節點執行
apt-get update && apt-get install -y haproxy
```

### 3.2 設定 haproxy.cfg

在所有 master 節點上建立相同的設定檔：

```bash
# 備份原始設定
cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak

# 寫入新設定
cat > /etc/haproxy/haproxy.cfg << 'EOF'
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    log         /dev/log local0
    log         /dev/log local1 notice
    chroot      /var/lib/haproxy
    stats       socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats       timeout 30s
    user        haproxy
    group       haproxy
    daemon

    # Default SSL material locations
    ca-base /etc/ssl/certs
    crt-base /etc/ssl/private

    # SSL settings
    ssl-default-bind-ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:RSA+AESGCM:RSA+AES:!aNULL:!MD5:!DSS
    ssl-default-bind-options no-sslv3

#---------------------------------------------------------------------
# Defaults settings
#---------------------------------------------------------------------
defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000ms
    timeout client  50000ms
    timeout server  50000ms
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

#---------------------------------------------------------------------
# HAProxy Stats Page（可選，用於監控）
#---------------------------------------------------------------------
listen stats
    bind *:8888
    mode  http
    stats enable
    stats uri /stats
    stats realm HAProxy\ Statistics
    stats auth admin:haproxy123
    stats refresh 10s

#---------------------------------------------------------------------
# Kubernetes API Server Frontend
#---------------------------------------------------------------------
frontend k8s-api-frontend
    bind *:6443
    mode tcp
    option tcplog
    default_backend k8s-api-backend

#---------------------------------------------------------------------
# Kubernetes API Server Backend
#---------------------------------------------------------------------
backend k8s-api-backend
    mode tcp
    option tcp-check
    balance roundrobin
    default-server inter 10s downinter 5s rise 2 fall 3 slowstart 60s maxconn 250 maxqueue 256 weight 100

    # Master 節點列表
    server k8s-master1 192.168.100.11:6443 check
    server k8s-master2 192.168.100.12:6443 check
    server k8s-master3 192.168.100.13:6443 check
EOF
```

### 3.3 驗證並啟動 HAProxy

```bash
# 驗證設定檔語法
haproxy -c -f /etc/haproxy/haproxy.cfg

# 啟動並設為開機自啟
systemctl enable haproxy
systemctl start haproxy
systemctl status haproxy

# 確認監聽 6443 port
ss -tlnp | grep 6443
```

---

## 4. Keepalived 安裝與設定

Keepalived 安裝在**所有 3 台 master 節點**，管理 VIP（192.168.100.10）的歸屬。

### 4.1 安裝 Keepalived

```bash
apt-get update && apt-get install -y keepalived
```

### 4.2 建立健康檢查腳本

在所有 master 節點上建立相同的健康檢查腳本：

```bash
mkdir -p /etc/keepalived

cat > /etc/keepalived/check_apiserver.sh << 'EOF'
#!/bin/bash
# HAProxy 健康檢查腳本
# 若 HAProxy 或 apiserver 停止服務，降低優先權讓 VIP 飄移

errorExit() {
    echo "*** $*" 1>&2
    exit 1
}

# 檢查本機 HAProxy 是否在監聽 6443
curl --silent --max-time 2 --insecure https://localhost:6443/ -o /dev/null || errorExit "Error GET https://localhost:6443/"

# 若 VIP 在本機，確認 apiserver 可透過 VIP 存取
if ip addr | grep -q "192.168.100.10"; then
    curl --silent --max-time 2 --insecure https://192.168.100.10:6443/ -o /dev/null || errorExit "Error GET https://192.168.100.10:6443/"
fi
EOF

chmod +x /etc/keepalived/check_apiserver.sh
```

### 4.3 設定 keepalived.conf（master1 - MASTER 角色）

```bash
# 確認網路介面名稱
ip link show | grep -E "^[0-9]"
# 本範例假設介面為 ens18，請依實際環境調整

# master1 設定（MASTER 角色，優先權最高）
cat > /etc/keepalived/keepalived.conf << 'EOF'
! Keepalived configuration for k8s-master1 (MASTER)

global_defs {
    router_id k8s-master1
    script_user root
    enable_script_security
}

vrrp_script check_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 3          # 每 3 秒執行一次
    timeout  2          # 超時 2 秒
    weight   -20        # 檢查失敗時，優先權降低 20
    fall     3          # 連續 3 次失敗才切換
    rise     2          # 連續 2 次成功才恢復
}

vrrp_instance VI_1 {
    state MASTER            # 初始角色：MASTER
    interface ens18         # 網路介面（依實際環境調整）
    virtual_router_id 51    # VRRP 群組 ID（同群組所有節點必須相同）
    priority 101            # 優先權（MASTER 最高）
    advert_int 1            # VRRP 廣播間隔（秒）
    authentication {
        auth_type PASS
        auth_pass k8s2024    # 認證密碼（同群組所有節點必須相同）
    }
    virtual_ipaddress {
        192.168.100.10/24   # VIP 位址
    }
    track_script {
        check_apiserver
    }
}
EOF
```

### 4.4 設定 keepalived.conf（master2 - BACKUP 角色）

```bash
# master2 設定（BACKUP 角色，優先權次高）
cat > /etc/keepalived/keepalived.conf << 'EOF'
! Keepalived configuration for k8s-master2 (BACKUP)

global_defs {
    router_id k8s-master2
    script_user root
    enable_script_security
}

vrrp_script check_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 3
    timeout  2
    weight   -20
    fall     3
    rise     2
}

vrrp_instance VI_1 {
    state BACKUP            # 初始角色：BACKUP
    interface ens18
    virtual_router_id 51
    priority 100            # 優先權次高
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass k8s2024
    }
    virtual_ipaddress {
        192.168.100.10/24
    }
    track_script {
        check_apiserver
    }
}
EOF
```

### 4.5 設定 keepalived.conf（master3 - BACKUP 角色）

```bash
# master3 設定（BACKUP 角色，優先權最低）
cat > /etc/keepalived/keepalived.conf << 'EOF'
! Keepalived configuration for k8s-master3 (BACKUP)

global_defs {
    router_id k8s-master3
    script_user root
    enable_script_security
}

vrrp_script check_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 3
    timeout  2
    weight   -20
    fall     3
    rise     2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens18
    virtual_router_id 51
    priority 99             # 優先權最低
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass k8s2024
    }
    virtual_ipaddress {
        192.168.100.10/24
    }
    track_script {
        check_apiserver
    }
}
EOF
```

### 4.6 啟動 Keepalived

```bash
# 在所有 master 節點執行
systemctl enable keepalived
systemctl start keepalived
systemctl status keepalived

# 驗證 VIP 已在 master1 上（master1 應有 192.168.100.10）
ip addr show ens18 | grep "192.168.100"
```

---

## 5. containerd 安裝與設定

在**所有 6 個節點**安裝 containerd。

### 5.1 安裝 containerd

```bash
# 新增 Docker 官方 GPG 金鑰
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

# 新增 Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null

# 更新並安裝 containerd
apt-get update
apt-get install -y containerd.io
```

### 5.2 設定 containerd

```bash
# 產生預設設定檔
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml

# 修改設定：啟用 SystemdCgroup（K8s 必須）
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# 確認修改成功
grep "SystemdCgroup" /etc/containerd/config.toml
# 預期輸出：SystemdCgroup = true

# 重啟 containerd
systemctl restart containerd
systemctl enable containerd
systemctl status containerd
```

### 5.3 安裝 crictl

```bash
# 安裝 crictl（用於除錯，類似 docker 指令）
VERSION="v1.29.0"
curl -L "https://github.com/kubernetes-sigs/cri-tools/releases/download/${VERSION}/crictl-${VERSION}-linux-amd64.tar.gz" | tar -C /usr/local/bin -xz

# 設定 crictl 使用 containerd
cat > /etc/crictl.yaml << 'EOF'
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 2
debug: false
EOF

# 驗證
crictl version
```

---

## 6. kubeadm / kubelet / kubectl 安裝

在**所有 6 個節點**安裝 kubeadm、kubelet、kubectl。

```bash
# 新增 Kubernetes apt repository（K8s 1.29）
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
  gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | \
  tee /etc/apt/sources.list.d/kubernetes.list

# 安裝
apt-get update
apt-get install -y kubelet kubeadm kubectl

# 鎖定版本，避免自動升級
apt-mark hold kubelet kubeadm kubectl

# 確認版本
kubeadm version
kubectl version --client
kubelet --version

# 啟動 kubelet（此時會不斷重啟，等 kubeadm init 完成後才正常）
systemctl enable kubelet
```

---

## 7. 初始化第一個 Master 節點

以下步驟**只在 master1** 執行。

### 7.1 建立 kubeadm 設定檔

```bash
cat > /root/kubeadm-config.yaml << 'EOF'
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: "1.29.0"
clusterName: "k8s-prod"

# 指向 VIP（所有 master 透過 VIP 存取 apiserver）
controlPlaneEndpoint: "192.168.100.10:6443"

# API Server 設定
apiServer:
  extraArgs:
    authorization-mode: "Node,RBAC"
    event-ttl: "1h"
  certSANs:
  - "192.168.100.10"    # VIP
  - "192.168.100.11"    # master1
  - "192.168.100.12"    # master2
  - "192.168.100.13"    # master3
  - "k8s-vip"
  - "k8s-master1"
  - "k8s-master2"
  - "k8s-master3"
  - "localhost"
  - "127.0.0.1"

# Controller Manager 設定
controllerManager:
  extraArgs:
    bind-address: "0.0.0.0"

# Scheduler 設定
scheduler:
  extraArgs:
    bind-address: "0.0.0.0"

# etcd 設定（stacked 模式，使用本機 etcd）
etcd:
  local:
    dataDir: /var/lib/etcd
    extraArgs:
      listen-metrics-urls: "http://0.0.0.0:2381"

# 網路設定（與 Calico 對應）
networking:
  serviceSubnet: "10.96.0.0/12"    # Service CIDR
  podSubnet: "10.244.0.0/16"       # Pod CIDR（Calico 使用）
  dnsDomain: "cluster.local"

---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "192.168.100.11"   # master1 的 IP
  bindPort: 6443

nodeRegistration:
  criSocket: "unix:///run/containerd/containerd.sock"
  name: "k8s-master1"
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/control-plane

---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: "systemd"
EOF
```

### 7.2 預先拉取映像（可選，加速初始化）

```bash
# 預先拉取所需映像
kubeadm config images pull --config /root/kubeadm-config.yaml

# 查看會拉取的映像清單
kubeadm config images list --config /root/kubeadm-config.yaml
```

### 7.3 執行 kubeadm init

```bash
# 初始化（上傳 certificate 到 Secret，供其他 master 節點使用）
kubeadm init \
  --config /root/kubeadm-config.yaml \
  --upload-certs \
  --v=5 \
  2>&1 | tee /root/kubeadm-init.log

# 注意：init 完成後，最後幾行會顯示 join 指令，請務必複製儲存
# 包含：
#   1. 其他 master 加入指令（含 --control-plane --certificate-key）
#   2. worker 加入指令（不含 --control-plane）
```

### 7.4 設定 kubectl 存取

```bash
# 設定 root 使用者的 kubeconfig
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# 驗證 kubectl 可以存取叢集
kubectl get nodes
kubectl get pods -n kube-system

# 若要讓一般使用者也能使用 kubectl
# su - ubuntu
# mkdir -p $HOME/.kube
# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
# sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 7.5 查看初始化輸出並記錄 Join 指令

```bash
# 查看初始化完整輸出
cat /root/kubeadm-init.log | tail -30

# 從 init 輸出取得 certificate-key（24 小時內有效）
# 範例輸出：
# You can now join any number of the control-plane node running the following command on each as root:
#
#   kubeadm join 192.168.100.10:6443 --token <TOKEN> \
#     --discovery-token-ca-cert-hash sha256:<HASH> \
#     --control-plane --certificate-key <CERT_KEY>
#
# Then you can join any number of worker nodes by running the following on each as root:
#
#   kubeadm join 192.168.100.10:6443 --token <TOKEN> \
#     --discovery-token-ca-cert-hash sha256:<HASH>
```

---

## 8. 加入其他 Master 節點

以下步驟分別在 **master2** 和 **master3** 執行。

### 8.1 重新上傳 Certificates（若 token 過期）

```bash
# 若 kubeadm init 輸出的 certificate-key 已過期（2 小時有效），
# 在 master1 重新上傳：
kubeadm init phase upload-certs --upload-certs

# 重新產生 token（24 小時有效）
kubeadm token create --print-join-command
```

### 8.2 master2 加入叢集

```bash
# 在 master2 執行（使用 kubeadm init 輸出的指令，替換實際 TOKEN、HASH、CERT_KEY）
kubeadm join 192.168.100.10:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --control-plane \
  --certificate-key <CERT_KEY> \
  --apiserver-advertise-address 192.168.100.12 \
  --cri-socket unix:///run/containerd/containerd.sock \
  2>&1 | tee /root/kubeadm-join-master2.log

# 設定 kubectl
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

### 8.3 master3 加入叢集

```bash
# 在 master3 執行
kubeadm join 192.168.100.10:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --control-plane \
  --certificate-key <CERT_KEY> \
  --apiserver-advertise-address 192.168.100.13 \
  --cri-socket unix:///run/containerd/containerd.sock \
  2>&1 | tee /root/kubeadm-join-master3.log

# 設定 kubectl
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

### 8.4 在 master1 確認所有 control plane 節點狀態

```bash
# 此時 CNI 尚未安裝，節點狀態為 NotReady，但應可看到所有 master 節點
kubectl get nodes
# 預期輸出：
# NAME          STATUS     ROLES           AGE   VERSION
# k8s-master1   NotReady   control-plane   10m   v1.29.0
# k8s-master2   NotReady   control-plane   5m    v1.29.0
# k8s-master3   NotReady   control-plane   2m    v1.29.0

# 確認 control plane Pod 都在運行
kubectl get pods -n kube-system
```

---

## 9. 加入 Worker 節點

以下步驟分別在 **worker1、worker2、worker3** 執行。

### 9.1 worker1 加入叢集

```bash
# 在 worker1 執行（使用 kubeadm init 輸出的 worker join 指令）
kubeadm join 192.168.100.10:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --cri-socket unix:///run/containerd/containerd.sock \
  2>&1 | tee /root/kubeadm-join-worker1.log
```

### 9.2 worker2 加入叢集

```bash
# 在 worker2 執行
kubeadm join 192.168.100.10:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --cri-socket unix:///run/containerd/containerd.sock \
  2>&1 | tee /root/kubeadm-join-worker2.log
```

### 9.3 worker3 加入叢集

```bash
# 在 worker3 執行
kubeadm join 192.168.100.10:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --cri-socket unix:///run/containerd/containerd.sock \
  2>&1 | tee /root/kubeadm-join-worker3.log
```

### 9.4 為 Worker 節點加上標籤

```bash
# 在 master1 執行，為 worker 節點加上 role 標籤
kubectl label node k8s-worker1 node-role.kubernetes.io/worker=worker
kubectl label node k8s-worker2 node-role.kubernetes.io/worker=worker
kubectl label node k8s-worker3 node-role.kubernetes.io/worker=worker

# 驗證（此時仍 NotReady，等 CNI 安裝後會變 Ready）
kubectl get nodes
```

---

## 10. Calico CNI 安裝

在 **master1** 上執行，安裝 Calico 網路插件。

### 10.1 安裝 Calico Operator

```bash
# 安裝 Calico Operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml

# 等待 operator 就緒
kubectl wait --namespace tigera-operator \
  --for=condition=ready pod \
  --selector=app=tigera-operator \
  --timeout=90s

kubectl get pods -n tigera-operator
```

### 10.2 建立 Calico 自訂資源設定

```bash
cat > /root/calico-custom-resources.yaml << 'EOF'
# Calico Installation 設定
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Calico 使用的網路模式
  calicoNetwork:
    ipPools:
    - name: default-ipv4-ippool
      blockSize: 26
      cidr: 10.244.0.0/16    # 與 kubeadm-config.yaml 中的 podSubnet 一致
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()

---
# Calico APIServer 設定（可選）
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
EOF

kubectl create -f /root/calico-custom-resources.yaml
```

### 10.3 等待 Calico 就緒

```bash
# 等待所有 Calico 元件就緒（約需 2-5 分鐘）
watch kubectl get pods -n calico-system

# 或用 wait 指令
kubectl wait --namespace calico-system \
  --for=condition=ready pod \
  --selector=k8s-app=calico-node \
  --timeout=180s

# 確認所有節點狀態變為 Ready
kubectl get nodes
# 預期輸出：所有節點 STATUS 為 Ready
```

### 10.4 安裝 calicoctl（選用，用於管理 Calico）

```bash
curl -L https://github.com/projectcalico/calico/releases/download/v3.27.0/calicoctl-linux-amd64 \
  -o /usr/local/bin/calicoctl
chmod +x /usr/local/bin/calicoctl

# 驗證
calicoctl version
calicoctl get nodes
```

---

## 11. 叢集驗證

### 11.1 所有節點狀態確認

```bash
# 確認所有節點為 Ready
kubectl get nodes -o wide
# 預期輸出：
# NAME          STATUS   ROLES           AGE   VERSION   INTERNAL-IP
# k8s-master1   Ready    control-plane   30m   v1.29.0   192.168.100.11
# k8s-master2   Ready    control-plane   25m   v1.29.0   192.168.100.12
# k8s-master3   Ready    control-plane   20m   v1.29.0   192.168.100.13
# k8s-worker1   Ready    worker          15m   v1.29.0   192.168.100.21
# k8s-worker2   Ready    worker          15m   v1.29.0   192.168.100.22
# k8s-worker3   Ready    worker          15m   v1.29.0   192.168.100.23

# 確認 kube-system Pod 都在運行
kubectl get pods -n kube-system
kubectl get pods -n calico-system
```

### 11.2 etcd 叢集健康檢查

```bash
# 在任一 master 節點執行
ETCDCTL_API=3 etcdctl \
  --endpoints=https://192.168.100.11:2379,https://192.168.100.12:2379,https://192.168.100.13:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list -w table

# 預期輸出（3 個成員，1 個 isLeader=true）：
# +------------------+---------+-------------+-----------------------------+...
# |        ID        | STATUS  |    NAME     |         PEER ADDRS          |...
# +------------------+---------+-------------+-----------------------------+...
# | ...              | started | k8s-master1 | https://192.168.100.11:2380 |...
# | ...              | started | k8s-master2 | https://192.168.100.12:2380 |...
# | ...              | started | k8s-master3 | https://192.168.100.13:2380 |...
# +------------------+---------+-------------+-----------------------------+...

# 檢查 etcd 健康
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://192.168.100.11:2379,https://192.168.100.12:2379,https://192.168.100.13:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
# 預期輸出：三個 endpoint 均顯示 healthy
```

### 11.3 HA 故障切換測試

```bash
# 測試 1：確認 VIP 目前在 master1
ip addr show ens18 | grep 192.168.100.10   # 在 master1 執行，應有輸出

# 測試 2：停止 master1 的 keepalived，觀察 VIP 飄移
# 在 master1 執行：
systemctl stop keepalived
# 等待幾秒後，在 master2 確認 VIP 已飄移：
# 在 master2 執行：
ip addr show ens18 | grep 192.168.100.10   # 應有輸出

# 測試 3：確認 kubectl 仍可使用（透過 VIP）
kubectl get nodes   # 應仍可正常執行

# 測試 4：恢復 master1
# 在 master1 執行：
systemctl start keepalived
```

### 11.4 部署測試應用驗證

```bash
# 部署 nginx 測試應用
kubectl create deployment test-nginx --image=nginx:1.25 --replicas=3

# 等待 Pod 就緒
kubectl wait --for=condition=ready pod \
  --selector=app=test-nginx --timeout=60s

# 確認 Pod 分散在不同 worker 節點
kubectl get pods -l app=test-nginx -o wide

# 建立 Service
kubectl expose deployment test-nginx --port=80 --type=NodePort

# 取得 NodePort
NODEPORT=$(kubectl get svc test-nginx -o jsonpath='{.spec.ports[0].nodePort}')
echo "NodePort: $NODEPORT"

# 測試存取
curl http://192.168.100.21:$NODEPORT
curl http://192.168.100.22:$NODEPORT
curl http://192.168.100.23:$NODEPORT

# 測試 Pod DNS 解析
kubectl run test-dns --image=busybox:1.28 --restart=Never -- \
  nslookup kubernetes.default.svc.cluster.local
kubectl logs test-dns

# 清理測試資源
kubectl delete deployment test-nginx
kubectl delete service test-nginx
kubectl delete pod test-dns
```

### 11.5 安裝 Metrics Server（監控用）

```bash
# 安裝 metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# 若叢集使用自簽憑證，需加上 --kubelet-insecure-tls 參數
kubectl patch deployment metrics-server -n kube-system \
  --type=json \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'

# 等待 metrics-server 就緒
kubectl wait --namespace kube-system \
  --for=condition=ready pod \
  --selector=k8s-app=metrics-server \
  --timeout=60s

# 驗證
kubectl top nodes
kubectl top pods -A
```

---

## 12. 常見問題排除

### 問題 1：節點狀態顯示 NotReady

**症狀**：`kubectl get nodes` 顯示節點為 `NotReady`

**原因與解法**：

```bash
# 步驟 1：查看節點詳細狀態
kubectl describe node <node-name> | grep -A 10 "Conditions:"

# 步驟 2：確認是否 CNI 未安裝（最常見原因）
kubectl get pods -n kube-system | grep -i calico
# 若 calico Pod 未運行，重新安裝 CNI

# 步驟 3：查看 kubelet 日誌
journalctl -u kubelet -n 50 --no-pager

# 步驟 4：確認 containerd 運行正常
systemctl status containerd
crictl ps

# 步驟 5：確認節點網路
ping 192.168.100.11   # 從其他節點測試連通性
```

---

### 問題 2：kubeadm init 失敗 - Port 已被佔用

**症狀**：`error execution phase preflight: [preflight] Some fatal errors occurred: [ERROR Port-6443]: Port 6443 is in use`

**原因**：HAProxy 已在監聽 6443 port（本 HA 架構的預期行為）。

**解法**：

```bash
# 忽略此前置檢查錯誤
kubeadm init \
  --config /root/kubeadm-config.yaml \
  --upload-certs \
  --ignore-preflight-errors=Port-6443 \
  2>&1 | tee /root/kubeadm-init.log

# 或在 kubeadm-config.yaml 的 InitConfiguration 中加入：
# skipPhases:
#   - preflight
```

---

### 問題 3：etcd 叢集不健康 / quorum 遺失

**症狀**：`kubectl` 指令回傳 `etcdserver: request timed out` 或 `etcdserver: no leader`

**原因**：超過半數 etcd 節點故障（3 節點叢集容忍 1 個故障）。

**解法**：

```bash
# 確認 etcd Pod 狀態
kubectl get pods -n kube-system -l component=etcd

# 查看 etcd 日誌
kubectl logs etcd-k8s-master1 -n kube-system --tail=50

# 若需要強制恢復單一節點（緊急情況）
# 1. 停止其他異常 etcd 節點
# 2. 修改存活節點的 etcd manifest（/etc/kubernetes/manifests/etcd.yaml）
# 加入 --force-new-cluster 參數（注意：這會破壞 etcd 叢集，需重新加入其他節點）
# 3. 操作完成後移除 --force-new-cluster

# etcd 備份（建議定期執行）
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# etcd 還原
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-20260101.db \
  --data-dir=/var/lib/etcd-restore
```

---

### 問題 4：kubeadm join token 過期

**症狀**：執行 `kubeadm join` 時出現 `token does not exist` 或 `token is expired`

**解法**：

```bash
# 在 master1 重新產生 token（24 小時有效）
kubeadm token create --print-join-command

# 確認現有 token
kubeadm token list

# 若需要加入 control-plane，需同時重新上傳 certificates（2 小時有效）
kubeadm init phase upload-certs --upload-certs
# 輸出中會顯示新的 --certificate-key

# 完整的新 master join 指令（組合 token + certificate-key）
kubeadm join 192.168.100.10:6443 \
  --token <NEW_TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --control-plane \
  --certificate-key <NEW_CERT_KEY>
```

---

### 問題 5：Pod 無法跨節點通訊（CNI 問題）

**症狀**：同節點的 Pod 可互通，但跨節點 Pod 無法通訊；或 Service ClusterIP 無法存取。

**診斷步驟**：

```bash
# 步驟 1：確認 Calico Pod 全部正常
kubectl get pods -n calico-system -o wide
# 所有 Pod 應為 Running，每個節點都有 calico-node Pod

# 步驟 2：確認 Pod CIDR 設定正確
kubectl get node k8s-worker1 -o jsonpath='{.spec.podCIDR}'
# 應顯示如 10.244.0.0/24

# 步驟 3：測試跨節點 Pod 通訊
kubectl run pod1 --image=busybox:1.28 --restart=Never -- sleep 3600
kubectl run pod2 --image=busybox:1.28 --restart=Never -- sleep 3600

# 確認在不同節點
kubectl get pods -o wide | grep -E "pod1|pod2"

# 取得 Pod IP
POD1_IP=$(kubectl get pod pod1 -o jsonpath='{.status.podIP}')
POD2_IP=$(kubectl get pod pod2 -o jsonpath='{.status.podIP}')

# 測試互通
kubectl exec pod1 -- ping -c 3 $POD2_IP

# 步驟 4：查看 Calico node 日誌
kubectl logs -n calico-system daemonset/calico-node --tail=50

# 步驟 5：確認 BGP peer 狀態（若使用 BGP 模式）
calicoctl node status

# 步驟 6：確認 podSubnet 與 Calico IPPOOL 一致
kubectl get ippools -o yaml

# 常見解法：若 podSubnet 與 Calico IPPOOL CIDR 不符，更新 IPPOOL：
# calicoctl apply -f - <<EOF
# apiVersion: projectcalico.org/v3
# kind: IPPool
# metadata:
#   name: default-ipv4-ippool
# spec:
#   cidr: 10.244.0.0/16
#   ipipMode: Never
#   vxlanMode: CrossSubnet
#   natOutgoing: true
# EOF

# 清理測試 Pod
kubectl delete pod pod1 pod2
```

---

### 問題 6：VIP 無法切換（Keepalived 故障）

**症狀**：master1 故障後，VIP 未飄移到 master2；或 VIP 頻繁切換（Flapping）。

**診斷與解法**：

```bash
# 查看 keepalived 日誌
journalctl -u keepalived -n 50 --no-pager

# 確認 VRRP 廣播可到達其他節點（避免防火牆擋住 VRRP multicast）
# VRRP 使用 IP Protocol 112，目的 IP 224.0.0.18
# 若有防火牆，需允許：
# iptables -I INPUT -d 224.0.0.0/8 -j ACCEPT
# iptables -I INPUT -p vrrp -j ACCEPT

# 手動測試健康檢查腳本
/etc/keepalived/check_apiserver.sh && echo "OK" || echo "FAIL"

# 若 VIP 頻繁切換，調整 weight 與 fall/rise 參數：
# 增大 fall 值（如 fall 5）減少誤判
# 增大 advert_int（如 2）減少廣播頻率

# 查看目前 VIP 在哪個節點
for host in 192.168.100.11 192.168.100.12 192.168.100.13; do
  echo -n "$host: "
  ssh $host "ip addr show ens18 | grep 192.168.100.10" 2>/dev/null || echo "(no VIP)"
done
```

---

## 附錄 A：etcd 定期備份腳本

```bash
cat > /usr/local/bin/etcd-backup.sh << 'EOF'
#!/bin/bash
# etcd 自動備份腳本

BACKUP_DIR="/backup/etcd"
DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="$BACKUP_DIR/etcd-$DATE.db"
RETAIN_DAYS=7

mkdir -p $BACKUP_DIR

ETCDCTL_API=3 etcdctl snapshot save $BACKUP_FILE \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

if [ $? -eq 0 ]; then
    echo "$(date): etcd backup successful: $BACKUP_FILE"
    # 驗證備份
    ETCDCTL_API=3 etcdctl snapshot status $BACKUP_FILE --write-out=table
    # 清理舊備份
    find $BACKUP_DIR -name "etcd-*.db" -mtime +$RETAIN_DAYS -delete
else
    echo "$(date): etcd backup FAILED!" >&2
    exit 1
fi
EOF

chmod +x /usr/local/bin/etcd-backup.sh

# 設定 cron job（每天凌晨 2 點執行）
echo "0 2 * * * root /usr/local/bin/etcd-backup.sh >> /var/log/etcd-backup.log 2>&1" \
  > /etc/cron.d/etcd-backup
```

---

## 附錄 B：叢集重置（緊急重新部署）

```bash
# 在所有節點執行：重置 kubeadm 設定
kubeadm reset --force

# 清理 CNI 設定
rm -rf /etc/cni/net.d

# 清理 iptables 規則
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X

# 清理 IPVS 規則
ipvsadm --clear

# 重置後可重新執行 kubeadm init / join 流程
```

---

*文件版本：1.0 | 更新日期：2026-04-29 | 適用 K8s 版本：1.29 | Ubuntu 24.04*
