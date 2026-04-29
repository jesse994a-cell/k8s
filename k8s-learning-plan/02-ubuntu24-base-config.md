# Ubuntu 24.04 基礎配置（所有節點）

> **適用版本**：Ubuntu 24.04 LTS（Noble Numbat）
> **適用節點**：所有 K8s 節點（master、worker）及 ceph 節點
> **目標**：完成系統初始化，安裝 containerd、kubeadm、kubelet、kubectl
> **撰寫日期**：2026-04-29

---

## 目錄

1. [初始化設定概覽](#1-初始化設定概覽)
2. [系統更新與基礎套件](#2-系統更新與基礎套件)
3. [時間同步設定](#3-時間同步設定)
4. [Hostname 與 /etc/hosts 配置](#4-hostname-與-etchosts-配置)
5. [網路設定（Netplan 靜態 IP）](#5-網路設定netplan-靜態-ip)
6. [Kernel 參數優化](#6-kernel-參數優化)
7. [Swap 關閉](#7-swap-關閉)
8. [AppArmor 說明](#8-apparmor-說明)
9. [防火牆設定（UFW）](#9-防火牆設定ufw)
10. [SSH 安全強化](#10-ssh-安全強化)
11. [Ansible 批次配置](#11-ansible-批次配置)
12. [containerd 安裝與設定](#12-containerd-安裝與設定)
13. [安裝 kubeadm、kubelet、kubectl](#13-安裝-kubeadmkubeletkubectl)
14. [安裝後驗證](#14-安裝後驗證)

---

## 1. 初始化設定概覽

### 1.1 節點清單

| Hostname | 角色 | IP |
|----------|------|-----|
| k8s-master1 | K8s Master | 192.168.20.11 |
| k8s-master2 | K8s Master | 192.168.20.12 |
| k8s-master3 | K8s Master | 192.168.20.13 |
| k8s-worker1 | K8s Worker | 192.168.30.11 |
| k8s-worker2 | K8s Worker | 192.168.30.12 |
| k8s-worker3 | K8s Worker | 192.168.30.13 |
| ceph-node1 | Ceph Storage | 192.168.40.11 |
| ceph-node2 | Ceph Storage | 192.168.40.12 |
| ceph-node3 | Ceph Storage | 192.168.40.13 |
| gitlab-server | GitLab | 192.168.50.11 |

### 1.2 執行方式說明

本文件指令分為兩類：
- **[管理機]**：在筆電或 Proxmox 主機上執行
- **[所有節點]** 或指定節點：SSH 進入各 VM 後執行

建議搭配 **Ansible 批次執行**（見第 11 節），可大幅節省時間。

---

## 2. 系統更新與基礎套件

```bash
# ── [所有節點] ──────────────────────────────────────────────

# 更新 apt 套件庫索引
sudo apt update

# 升級已安裝的套件
sudo apt upgrade -y

# 安裝基礎工具套件
sudo apt install -y \
  vim \
  curl \
  wget \
  git \
  htop \
  iotop \
  iftop \
  net-tools \
  dnsutils \
  iputils-ping \
  traceroute \
  tcpdump \
  bash-completion \
  jq \
  unzip \
  rsync \
  lsof \
  sysstat \
  lvm2 \
  open-iscsi \
  nfs-common \
  socat \
  conntrack \
  ipset \
  ipvsadm \
  ethtool

# 清理不必要的套件
sudo apt autoremove -y
sudo apt autoclean

echo "基礎套件安裝完成"
```

---

## 3. 時間同步設定

> **重要**：K8s 叢集對時間同步極為敏感，節點時間差異超過 2 秒可能導致 etcd 選舉失敗

```bash
# ── [所有節點] ──────────────────────────────────────────────

# 安裝 chrony（Ubuntu 24.04 預設使用 systemd-timesyncd，建議換用 chrony）
sudo apt install -y chrony

# 停用 systemd-timesyncd（避免衝突）
sudo systemctl stop systemd-timesyncd
sudo systemctl disable systemd-timesyncd

# 備份原始設定
sudo cp /etc/chrony/chrony.conf /etc/chrony/chrony.conf.bak

# 設定 chrony（使用台灣 NTP 伺服器）
sudo tee /etc/chrony/chrony.conf << 'EOF'
# NTP 伺服器設定（台灣 NTP pool）
pool tw.pool.ntp.org iburst maxsources 4
pool time.google.com iburst
pool time.cloudflare.com iburst

# 記錄時鐘漂移
driftfile /var/lib/chrony/drift

# 首次同步時允許大幅度時間校正
makestep 1.0 3

# RTC 同步
rtcsync

# 日誌設定
logdir /var/log/chrony
log measurements statistics tracking
EOF

# 啟動並啟用 chrony
sudo systemctl enable chrony
sudo systemctl restart chrony

# 強制立即同步
sudo chronyc -a makestep

# 設定時區為台北
sudo timedatectl set-timezone Asia/Taipei

# 等待幾秒後確認同步狀態
sleep 3
chronyc tracking
# 預期：Reference ID 顯示 NTP server，System time offset < 0.01 seconds

# 確認時區
timedatectl status
# 預期：Time zone: Asia/Taipei (CST, +0800)
```

### 3.1 驗證時間同步

```bash
# 查看 NTP 同步來源
chronyc sources -v

# 確認各節點時間一致（在管理機執行，比較所有節點時間）
for ip in 192.168.20.11 192.168.20.12 192.168.20.13 \
          192.168.30.11 192.168.30.12 192.168.30.13; do
    echo -n "$ip: "
    ssh -i ~/.ssh/k8s_admin_key ubuntu@$ip "date '+%Y-%m-%d %H:%M:%S'" 2>/dev/null
done
```

---

## 4. Hostname 與 /etc/hosts 配置

### 4.1 設定各節點 Hostname

```bash
# 在各節點上分別執行對應指令：

# k8s-master1 上執行：
sudo hostnamectl set-hostname k8s-master1

# k8s-master2 上執行：
sudo hostnamectl set-hostname k8s-master2

# k8s-master3 上執行：
sudo hostnamectl set-hostname k8s-master3

# k8s-worker1 上執行：
sudo hostnamectl set-hostname k8s-worker1

# k8s-worker2 上執行：
sudo hostnamectl set-hostname k8s-worker2

# k8s-worker3 上執行：
sudo hostnamectl set-hostname k8s-worker3

# ceph-node1 上執行：
sudo hostnamectl set-hostname ceph-node1

# ceph-node2 上執行：
sudo hostnamectl set-hostname ceph-node2

# ceph-node3 上執行：
sudo hostnamectl set-hostname ceph-node3

# gitlab-server 上執行：
sudo hostnamectl set-hostname gitlab-server

# 確認 hostname
hostname
hostnamectl status
```

### 4.2 使用 Ansible 批次設定 Hostname

```bash
# [管理機] 使用 Ansible 一次設定所有節點 hostname
ansible all -i ~/k8s-ansible/inventory/hosts.ini \
  -m hostname \
  -a "name={{ inventory_hostname }}" \
  --become
```

### 4.3 配置 /etc/hosts（所有節點相同）

```bash
# ── [所有節點] ──────────────────────────────────────────────

# 設定 /etc/hosts（包含所有叢集節點）
# 先取得目前 hostname
CURRENT_HOST=$(hostname)

sudo tee /etc/hosts << EOF
# Loopback
127.0.0.1   localhost
127.0.1.1   ${CURRENT_HOST}

# IPv6
::1         localhost ip6-localhost ip6-loopback
ff02::1     ip6-allnodes
ff02::2     ip6-allrouters

# ── K8s Control Plane ────────────────────────────────────────
192.168.20.11   k8s-master1  k8s-master1.cluster.local
192.168.20.12   k8s-master2  k8s-master2.cluster.local
192.168.20.13   k8s-master3  k8s-master3.cluster.local
192.168.20.100  k8s-api      k8s-api.cluster.local

# ── K8s Workers ──────────────────────────────────────────────
192.168.30.11   k8s-worker1  k8s-worker1.cluster.local
192.168.30.12   k8s-worker2  k8s-worker2.cluster.local
192.168.30.13   k8s-worker3  k8s-worker3.cluster.local

# ── Ceph Storage ─────────────────────────────────────────────
192.168.40.11   ceph-node1   ceph-node1.storage.local
192.168.40.12   ceph-node2   ceph-node2.storage.local
192.168.40.13   ceph-node3   ceph-node3.storage.local

# ── CI/CD ────────────────────────────────────────────────────
192.168.50.11   gitlab-server  gitlab.yourdomain.com
EOF

# 驗證 hosts 解析
ping -c 2 k8s-master1
getent hosts k8s-api
```

---

## 5. 網路設定（Netplan 靜態 IP）

### 5.1 查看目前網路介面

```bash
# 查看目前網路介面名稱
ip link show
# Ubuntu 24.04 VM 通常為 eth0 或 ens3

# 查看目前 Netplan 設定
ls /etc/netplan/
cat /etc/netplan/*.yaml
```

### 5.2 Master Node 靜態 IP 設定（k8s-master1）

```bash
# ── [k8s-master1] ─────────────────────────────────────────

# 備份原始設定
sudo cp /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.bak 2>/dev/null || true

# 若 cloud-init 已正確設定 IP，此步驟可跳過
# 需要修改時才執行：
sudo tee /etc/netplan/50-cloud-init.yaml << 'EOF'
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: false
      dhcp6: false
      addresses:
        - 192.168.20.11/24
      routes:
        - to: default
          via: 192.168.20.1
          metric: 100
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
        search:
          - cluster.local
          - yourdomain.com
EOF

# 套用設定
sudo netplan --debug try
sudo netplan apply

# 驗證
ip addr show eth0
ip route show
```

### 5.3 Worker Node 設定（k8s-worker1）

```bash
sudo tee /etc/netplan/50-cloud-init.yaml << 'EOF'
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 192.168.30.11/24
      routes:
        - to: default
          via: 192.168.30.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
        search:
          - cluster.local
          - yourdomain.com
EOF
sudo netplan apply
```

### 5.4 Ceph Node 雙網路介面設定（ceph-node1）

```bash
# Ceph node 需要兩個網路介面：
# eth0 → Ceph public network（Client 存取，192.168.40.x）
# eth1 → Ceph cluster network（OSD replication 內部流量，192.168.41.x）
sudo tee /etc/netplan/50-cloud-init.yaml << 'EOF'
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 192.168.40.11/24
      routes:
        - to: default
          via: 192.168.40.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
    eth1:
      dhcp4: false
      addresses:
        - 192.168.41.11/24
      # eth1 不設 gateway，為 Ceph 內部 replication 專用
EOF
sudo netplan apply
```

---

## 6. Kernel 參數優化

### 6.1 載入必要的 Kernel 模組

```bash
# ── [所有 K8s 節點（master + worker）] ──────────────────────

# 建立模組載入設定（開機時自動載入）
sudo tee /etc/modules-load.d/k8s.conf << 'EOF'
# Kubernetes 必要 kernel 模組

# overlay filesystem（containerd 需要）
overlay

# bridge netfilter（iptables 處理 bridge 流量）
br_netfilter

# IPVS 模組（kube-proxy IPVS 模式）
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh

# conntrack（連線追蹤）
nf_conntrack
EOF

# 立即載入所有模組（不需要重啟）
sudo modprobe overlay
sudo modprobe br_netfilter
sudo modprobe ip_vs
sudo modprobe ip_vs_rr
sudo modprobe ip_vs_wrr
sudo modprobe ip_vs_sh
sudo modprobe nf_conntrack

# 確認模組已載入
lsmod | grep -E "overlay|br_netfilter|ip_vs|nf_conntrack"
# 應有輸出，表示模組已載入
```

### 6.2 Sysctl 參數設定

```bash
# ── [所有 K8s 節點] ──────────────────────────────────────────

sudo tee /etc/sysctl.d/99-kubernetes.conf << 'EOF'
# ============================================================
# Kubernetes 叢集 Kernel 參數優化
# ============================================================

# ── K8s 核心必要參數 ─────────────────────────────────────────

# 讓 bridge 流量通過 iptables 處理（K8s Service 和 NetworkPolicy 需要）
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1

# 啟用 IPv4 forwarding（Pod 路由必須）
net.ipv4.ip_forward = 1

# 啟用 IPv6 forwarding（若使用 IPv6）
net.ipv6.conf.all.forwarding = 1

# ── 連線追蹤（高併發場景）────────────────────────────────────

# 增加 conntrack 表大小（預設 131072，高併發不夠）
net.netfilter.nf_conntrack_max = 1048576

# 增加 conntrack hashtable
net.netfilter.nf_conntrack_buckets = 262144

# 縮短 TIME_WAIT 超時（高併發 HTTP）
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 30

# ── TCP 效能優化 ─────────────────────────────────────────────

# 啟用 TCP BBR 擁塞控制（kernel 4.9+，Ubuntu 24.04 支援）
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr

# 增大 TCP 緩衝區（高吞吐量）
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864

# 增大 socket 接收佇列（高併發時防止丟包）
net.core.netdev_max_backlog = 65536
net.core.somaxconn = 65535

# 增大 syn backlog（高併發服務）
net.ipv4.tcp_max_syn_backlog = 65535

# 啟用 TIME_WAIT socket 重用（高併發短連線）
net.ipv4.tcp_tw_reuse = 1

# 擴展本地埠範圍
net.ipv4.ip_local_port_range = 1024 65535

# TCP keepalive（偵測死連線）
net.ipv4.tcp_keepalive_time   = 600
net.ipv4.tcp_keepalive_intvl  = 60
net.ipv4.tcp_keepalive_probes = 10

# ── 記憶體與 VM 參數 ─────────────────────────────────────────

# 關閉 swappiness（K8s 不允許使用 swap）
vm.swappiness = 0

# 允許記憶體 overcommit（容器場景）
vm.overcommit_memory = 1

# dirty page 比例（提升寫入效能）
vm.dirty_ratio = 60
vm.dirty_background_ratio = 10

# ── 檔案系統參數 ─────────────────────────────────────────────

# 增大 inotify 監看數量（K8s 監看設定檔需要）
fs.inotify.max_user_watches   = 1048576
fs.inotify.max_user_instances = 8192

# 增大檔案描述符限制（高併發容器）
fs.file-max = 2097152
fs.nr_open  = 2097152

# ── 核心限制 ─────────────────────────────────────────────────

# 增大 PID 最大數（大量容器場景）
kernel.pid_max = 4194304

# ── 安全性參數 ───────────────────────────────────────────────

# 防止 ICMP redirect 攻擊
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects   = 0

# 停用 rp_filter（K8s Pod 網路需要）
net.ipv4.conf.all.rp_filter     = 0
net.ipv4.conf.default.rp_filter = 0
EOF

# 立即套用所有 sysctl 設定
sudo sysctl --system

# 確認關鍵參數
echo "=== 驗證關鍵 Kernel 參數 ==="
echo "bridge-nf-call-iptables:  $(sysctl -n net.bridge.bridge-nf-call-iptables)"
echo "ip_forward:               $(sysctl -n net.ipv4.ip_forward)"
echo "vm.swappiness:            $(sysctl -n vm.swappiness)"
echo "inotify.max_user_watches: $(sysctl -n fs.inotify.max_user_watches)"
echo "nf_conntrack_max:         $(sysctl -n net.netfilter.nf_conntrack_max)"
echo "tcp_congestion_control:   $(sysctl -n net.ipv4.tcp_congestion_control)"
```

### 6.3 設定系統資源限制

```bash
# 設定 systemd 層級的資源限制
sudo mkdir -p /etc/systemd/system.conf.d

sudo tee /etc/systemd/system.conf.d/k8s-limits.conf << 'EOF'
[Manager]
DefaultLimitNOFILE=1048576
DefaultLimitNPROC=1048576
DefaultLimitCORE=infinity
DefaultTasksMax=infinity
EOF

# 設定 PAM ulimit（SSH 連線時生效）
sudo tee -a /etc/security/limits.conf << 'EOF'
# K8s 叢集資源限制
*    soft nofile 1048576
*    hard nofile 1048576
*    soft nproc  1048576
*    hard nproc  1048576
root soft nofile 1048576
root hard nofile 1048576
EOF

# 重新載入 systemd 設定
sudo systemctl daemon-reload
```

---

## 7. Swap 關閉

> **重要**：Kubernetes 要求關閉 swap，否則 kubelet 預設會拒絕啟動

```bash
# ── [所有 K8s 節點] ──────────────────────────────────────────

# 立即關閉 swap
sudo swapoff -a

# 確認 swap 已關閉
free -h
# 預期：Swap: 0B total（全為 0）

# 永久關閉：從 /etc/fstab 移除 swap 設定
sudo sed -i '/\bswap\b/s/^/#/' /etc/fstab

# 確認 fstab 中 swap 已被註解
grep swap /etc/fstab
# 預期：所有 swap 行前都有 # 號

# 停用 systemd swap unit（Ubuntu 24.04 可能有）
sudo systemctl mask swap.target 2>/dev/null || true

# 再次確認 swap 關閉
swapon --show
# 預期：無輸出
```

---

## 8. AppArmor 說明

### 8.1 Ubuntu 24.04 與 AppArmor

Ubuntu 24.04 預設啟用 AppArmor（Linux Mandatory Access Control）。

```bash
# 確認 AppArmor 狀態
sudo apparmor_status | head -20

# 確認 kernel 層面已啟用
cat /sys/module/apparmor/parameters/enabled
# 預期：Y
```

### 8.2 K8s 與 AppArmor 的關係

```bash
# Kubernetes 1.30+ 原生支援 AppArmor，不需要修改
# containerd 和 K8s 都可以與 AppArmor 共存

# 若遇到 AppArmor 導致容器問題，可查看 audit log
sudo journalctl -k | grep apparmor | tail -20

# 不建議完全停用 AppArmor（會降低安全性）
# 若特定 Pod 需要放寬限制，在 Pod spec 中設定：
# securityContext:
#   appArmorProfile:
#     type: Unconfined
```

---

## 9. 防火牆設定（UFW）

### 9.1 Master Node 防火牆規則

```bash
# ── [k8s-master1、k8s-master2、k8s-master3] ─────────────────

sudo ufw --force reset
sudo ufw default deny incoming
sudo ufw default allow outgoing

# SSH（必須先開放，否則會鎖住自己）
sudo ufw allow 22/tcp comment "SSH"

# Kubernetes API Server（所有人都需要能存取）
sudo ufw allow 6443/tcp comment "K8s API Server"

# etcd（僅 master 節點之間）
sudo ufw allow from 192.168.20.0/24 to any port 2379:2380 proto tcp \
  comment "etcd peer communication"

# Kubelet API
sudo ufw allow 10250/tcp comment "Kubelet API"

# CNI（Cilium）
sudo ufw allow 8472/udp comment "Cilium VXLAN"
sudo ufw allow 4240/tcp comment "Cilium Health"
sudo ufw allow 4244/tcp comment "Cilium Hubble"

# ICMP（允許 ping，健康檢查需要）
sudo ufw allow proto icmp comment "ICMP"

# 允許 Pod 和 Service 網路（K8s 內部流量）
sudo ufw allow from 10.244.0.0/16 comment "Pod Network"
sudo ufw allow from 10.96.0.0/12  comment "Service Network"

# 啟用 UFW
sudo ufw --force enable

# 確認規則
sudo ufw status verbose
```

### 9.2 Worker Node 防火牆規則

```bash
# ── [k8s-worker1、k8s-worker2、k8s-worker3] ─────────────────

sudo ufw --force reset
sudo ufw default deny incoming
sudo ufw default allow outgoing

# SSH
sudo ufw allow 22/tcp comment "SSH"

# Kubelet API（Master 需要呼叫 Worker）
sudo ufw allow from 192.168.20.0/24 to any port 10250 proto tcp \
  comment "Kubelet API from master"

# NodePort Services（應用程式對外服務）
sudo ufw allow 30000:32767/tcp comment "NodePort TCP"
sudo ufw allow 30000:32767/udp comment "NodePort UDP"

# CNI（Cilium）
sudo ufw allow 8472/udp comment "Cilium VXLAN"
sudo ufw allow 4240/tcp comment "Cilium Health"
sudo ufw allow 4244/tcp comment "Cilium Hubble"

# ICMP
sudo ufw allow proto icmp comment "ICMP"

# Pod 和 Service 網路
sudo ufw allow from 10.244.0.0/16 comment "Pod Network"
sudo ufw allow from 10.96.0.0/12  comment "Service Network"

sudo ufw --force enable
sudo ufw status verbose
```

### 9.3 Ceph Node 防火牆規則

```bash
# ── [ceph-node1、ceph-node2、ceph-node3] ────────────────────

sudo ufw --force reset
sudo ufw default deny incoming
sudo ufw default allow outgoing

# SSH
sudo ufw allow 22/tcp comment "SSH"

# Ceph Monitor（MON）
sudo ufw allow from 192.168.40.0/24 to any port 6789 proto tcp comment "Ceph MON"
sudo ufw allow from 192.168.20.0/24 to any port 6789 proto tcp comment "Ceph MON from master"
sudo ufw allow from 192.168.30.0/24 to any port 6789 proto tcp comment "Ceph MON from worker"

# Ceph OSD 通訊
sudo ufw allow from 192.168.40.0/24 to any port 6800:7300 proto tcp comment "Ceph OSD"
sudo ufw allow from 192.168.41.0/24 to any port 6800:7300 proto tcp comment "Ceph OSD Cluster"

# Ceph Manager Dashboard
sudo ufw allow from 192.168.20.0/24 to any port 8080 proto tcp comment "Ceph Dashboard HTTP"
sudo ufw allow from 192.168.20.0/24 to any port 8443 proto tcp comment "Ceph Dashboard HTTPS"

# ICMP
sudo ufw allow proto icmp comment "ICMP"

sudo ufw --force enable
sudo ufw status verbose
```

### 9.4 所有 K8s 相關 Port 整理

```
Control Plane Ports：
  TCP 6443     → Kubernetes API Server
  TCP 2379-2380 → etcd（master 之間）
  TCP 10250    → Kubelet API
  TCP 10257    → kube-controller-manager
  TCP 10259    → kube-scheduler

Worker Node Ports：
  TCP 10250    → Kubelet API
  TCP 30000-32767 → NodePort Services

CNI（Cilium）Ports：
  UDP 8472     → VXLAN overlay
  TCP 4240     → Cilium health check
  TCP 4244     → Hubble observability
  TCP 4245     → Hubble relay

Ceph Ports：
  TCP 6789     → Monitor（MON）
  TCP 6800-7300 → OSD 通訊
  TCP 8080/8443 → Dashboard
```

---

## 10. SSH 安全強化

```bash
# ── [所有節點] ──────────────────────────────────────────────

# 備份原始設定
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak

# 建立安全的 SSH 設定
sudo tee /etc/ssh/sshd_config << 'EOF'
# ============================================================
# SSH 安全強化設定
# ============================================================

Port 22
AddressFamily inet
ListenAddress 0.0.0.0

# 主機金鑰
HostKey /etc/ssh/ssh_host_ed25519_key
HostKey /etc/ssh/ssh_host_rsa_key
HostKey /etc/ssh/ssh_host_ecdsa_key

# 加密演算法（僅允許強加密）
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group14-sha256
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com

# 認證設定
LoginGraceTime 30
PermitRootLogin no           # 禁止 root 直接登入
StrictModes yes
MaxAuthTries 3               # 最多 3 次嘗試
MaxSessions 10

# 公鑰認證
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys

# 禁止密碼登入（僅允許公鑰）
PasswordAuthentication no
PermitEmptyPasswords no
ChallengeResponseAuthentication no

# 停用不需要的功能
KerberosAuthentication no
GSSAPIAuthentication no
X11Forwarding no

# 啟用 PAM
UsePAM yes

# 連線保持
ClientAliveInterval 300
ClientAliveCountMax 3
TCPKeepAlive yes

# 僅允許 ubuntu 使用者 SSH
AllowUsers ubuntu

# 登入橫幅
Banner /etc/ssh/banner

# 日誌等級
LogLevel VERBOSE
SyslogFacility AUTH

# SFTP subsystem
Subsystem sftp /usr/lib/openssh/sftp-server
EOF

# 建立登入橫幅
sudo tee /etc/ssh/banner << 'EOF'
*******************************************************************************
                    AUTHORIZED ACCESS ONLY
  This system is restricted to authorized K8s cluster administrators.
  All activities may be monitored and recorded.
*******************************************************************************
EOF

# 測試設定檔語法（無輸出表示正確）
sudo sshd -t

# 重啟 SSH 服務
sudo systemctl restart sshd
sudo systemctl status sshd

echo "SSH 安全強化完成"
echo "重要：請在關閉目前連線前，用新終端測試 SSH 是否仍可正常連線！"
```

---

## 11. Ansible 批次配置

### 11.1 安裝 Ansible（管理機）

```bash
# ── [管理機] ─────────────────────────────────────────────────

# Ubuntu/Debian 安裝
sudo apt update
sudo apt install -y ansible python3-pip

# 確認安裝
ansible --version
```

### 11.2 建立 Ansible Inventory

```bash
mkdir -p ~/k8s-ansible/{inventory,playbooks,templates,group_vars}

cat > ~/k8s-ansible/inventory/hosts.ini << 'EOF'
[k8s_master]
k8s-master1 ansible_host=192.168.20.11
k8s-master2 ansible_host=192.168.20.12
k8s-master3 ansible_host=192.168.20.13

[k8s_worker]
k8s-worker1 ansible_host=192.168.30.11
k8s-worker2 ansible_host=192.168.30.12
k8s-worker3 ansible_host=192.168.30.13

[ceph]
ceph-node1  ansible_host=192.168.40.11
ceph-node2  ansible_host=192.168.40.12
ceph-node3  ansible_host=192.168.40.13

[gitlab]
gitlab-server ansible_host=192.168.50.11

[k8s_nodes:children]
k8s_master
k8s_worker

[all_nodes:children]
k8s_nodes
ceph
gitlab

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/k8s_admin_key
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
ansible_python_interpreter=/usr/bin/python3
EOF
```

### 11.3 Ansible Playbook - 基礎設定

```yaml
# ~/k8s-ansible/playbooks/01-base-config.yaml
---
- name: Ubuntu 24.04 基礎配置
  hosts: all_nodes
  become: true

  vars:
    timezone: "Asia/Taipei"

  tasks:
    - name: 更新 apt 快取
      apt:
        update_cache: true
        cache_valid_time: 3600

    - name: 安裝基礎套件
      apt:
        name:
          - vim
          - curl
          - wget
          - git
          - htop
          - net-tools
          - dnsutils
          - bash-completion
          - jq
          - socat
          - conntrack
          - ipset
          - ipvsadm
          - lvm2
          - nfs-common
        state: present

    - name: 安裝 chrony
      apt:
        name: chrony
        state: present

    - name: 停用 systemd-timesyncd
      systemd:
        name: systemd-timesyncd
        state: stopped
        enabled: false
      ignore_errors: true

    - name: 設定時區
      timezone:
        name: "{{ timezone }}"

    - name: 啟動 chrony
      systemd:
        name: chrony
        state: started
        enabled: true

    - name: 設定 hostname
      hostname:
        name: "{{ inventory_hostname }}"

    - name: 設定 kernel 模組自動載入
      copy:
        content: |
          overlay
          br_netfilter
          ip_vs
          ip_vs_rr
          ip_vs_wrr
          ip_vs_sh
          nf_conntrack
        dest: /etc/modules-load.d/k8s.conf

    - name: 載入 kernel 模組
      modprobe:
        name: "{{ item }}"
        state: present
      loop:
        - overlay
        - br_netfilter
        - ip_vs
        - ip_vs_rr
        - ip_vs_wrr
        - ip_vs_sh
        - nf_conntrack

    - name: 設定 sysctl 參數
      sysctl:
        name: "{{ item.name }}"
        value: "{{ item.value }}"
        sysctl_file: /etc/sysctl.d/99-kubernetes.conf
        reload: true
      loop:
        - { name: "net.bridge.bridge-nf-call-iptables",  value: "1" }
        - { name: "net.bridge.bridge-nf-call-ip6tables", value: "1" }
        - { name: "net.ipv4.ip_forward",                 value: "1" }
        - { name: "vm.swappiness",                       value: "0" }
        - { name: "fs.inotify.max_user_watches",         value: "1048576" }
        - { name: "net.netfilter.nf_conntrack_max",      value: "1048576" }

    - name: 立即關閉 swap
      command: swapoff -a

    - name: 從 fstab 移除 swap
      replace:
        path: /etc/fstab
        regexp: '^([^#].*\s+swap\s+.*)$'
        replace: '# \1'
```

### 11.4 Ansible Playbook - 安裝 K8s 工具

```yaml
# ~/k8s-ansible/playbooks/02-install-k8s-tools.yaml
---
- name: 安裝 containerd 與 K8s 工具
  hosts: k8s_nodes
  become: true

  tasks:
    - name: 安裝 containerd 依賴
      apt:
        name:
          - apt-transport-https
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
        state: present

    - name: 建立 apt keyrings 目錄
      file:
        path: /etc/apt/keyrings
        state: directory
        mode: '0755'

    - name: 下載 Docker GPG key
      shell: |
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
        gpg --dearmor -o /etc/apt/keyrings/docker.gpg
      args:
        creates: /etc/apt/keyrings/docker.gpg

    - name: 新增 Docker apt repo
      apt_repository:
        repo: "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu noble stable"
        state: present
        filename: docker

    - name: 安裝 containerd
      apt:
        name: containerd.io
        state: present
        update_cache: true

    - name: 生成 containerd 預設設定
      shell: containerd config default > /etc/containerd/config.toml
      args:
        creates: /etc/containerd/config.toml

    - name: 啟用 SystemdCgroup
      replace:
        path: /etc/containerd/config.toml
        regexp: 'SystemdCgroup = false'
        replace: 'SystemdCgroup = true'
      notify: restart containerd

    - name: 啟動 containerd
      systemd:
        name: containerd
        state: started
        enabled: true

    - name: 下載 Kubernetes GPG key
      shell: |
        curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
        gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
      args:
        creates: /etc/apt/keyrings/kubernetes-apt-keyring.gpg

    - name: 新增 Kubernetes apt repo
      apt_repository:
        repo: "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /"
        state: present
        filename: kubernetes

    - name: 安裝 kubeadm、kubelet、kubectl
      apt:
        name:
          - kubelet
          - kubeadm
          - kubectl
        state: present
        update_cache: true

    - name: 鎖定 K8s 套件版本
      dpkg_selections:
        name: "{{ item }}"
        selection: hold
      loop:
        - kubelet
        - kubeadm
        - kubectl

    - name: 啟用 kubelet
      systemd:
        name: kubelet
        enabled: true

  handlers:
    - name: restart containerd
      systemd:
        name: containerd
        state: restarted
```

### 11.5 執行 Playbooks

```bash
# ── [管理機] ─────────────────────────────────────────────────

cd ~/k8s-ansible

# 測試連線
ansible all -i inventory/hosts.ini -m ping

# 執行基礎設定
ansible-playbook -i inventory/hosts.ini playbooks/01-base-config.yaml -v

# 執行 K8s 工具安裝（僅 K8s 節點）
ansible-playbook -i inventory/hosts.ini playbooks/02-install-k8s-tools.yaml -v

# 乾跑模式（不實際執行）
ansible-playbook -i inventory/hosts.ini playbooks/01-base-config.yaml --check

# 只在特定節點執行
ansible-playbook -i inventory/hosts.ini playbooks/01-base-config.yaml \
  --limit k8s_master -v
```

---

## 12. containerd 安裝與設定

### 12.1 手動安裝步驟

```bash
# ── [所有 K8s 節點] ──────────────────────────────────────────

# 安裝必要依賴
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release

# 建立 keyrings 目錄
sudo install -m 0755 -d /etc/apt/keyrings

# 下載 Docker GPG 金鑰
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 新增 Docker apt repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安裝 containerd
sudo apt update
sudo apt install -y containerd.io

# 確認版本
containerd --version
```

### 12.2 設定 containerd

```bash
# 建立設定目錄
sudo mkdir -p /etc/containerd

# 生成預設設定
sudo containerd config default | sudo tee /etc/containerd/config.toml

# 啟用 SystemdCgroup（重要！K8s 使用 systemd cgroup driver）
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# 確認修改成功
grep SystemdCgroup /etc/containerd/config.toml
# 預期：SystemdCgroup = true

# 設定 pause container 版本（與 kubeadm 版本對應）
sudo sed -i 's|sandbox_image = "registry.k8s.io/pause:.*"|sandbox_image = "registry.k8s.io/pause:3.9"|' \
  /etc/containerd/config.toml

# 重啟 containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd
```

### 12.3 驗證 containerd

```bash
# 確認 containerd socket
ls -la /run/containerd/containerd.sock

# 確認版本
sudo ctr version

# 拉取測試映像
sudo ctr images pull docker.io/library/hello-world:latest

# 執行測試容器
sudo ctr run --rm docker.io/library/hello-world:latest test-container

# 清理測試映像
sudo ctr images rm docker.io/library/hello-world:latest

echo "containerd 驗證完成"
```

---

## 13. 安裝 kubeadm、kubelet、kubectl

### 13.1 設定 Kubernetes apt Repository

```bash
# ── [所有 K8s 節點] ──────────────────────────────────────────

# 建立 keyrings 目錄
sudo mkdir -p /etc/apt/keyrings

# 下載 Kubernetes apt 簽名金鑰（v1.30）
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# 新增 Kubernetes apt repo
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

# 更新 apt
sudo apt update
```

### 13.2 安裝 K8s 工具

```bash
# 查看可用版本（確認 repo 正確）
apt-cache madison kubeadm | head -5

# 安裝最新 v1.30.x
sudo apt install -y kubelet kubeadm kubectl

# 鎖定版本（防止 apt upgrade 自動升級）
sudo apt-mark hold kubelet kubeadm kubectl

# 確認安裝版本
kubeadm version
kubectl version --client
kubelet --version

# 啟用 kubelet 服務（目前啟動會失敗，這是正常的，kubeadm init 後才會正常）
sudo systemctl enable kubelet

echo "K8s 工具安裝完成"
```

### 13.3 安裝 Helm

```bash
# ── [管理機 或 master1] ───────────────────────────────────────

# 安裝 helm（K8s 套件管理工具）
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 確認安裝
helm version

# 新增常用 helm repo
helm repo add stable         https://charts.helm.sh/stable
helm repo add bitnami        https://charts.bitnami.com/bitnami
helm repo add ingress-nginx  https://kubernetes.github.io/ingress-nginx
helm repo add cert-manager   https://charts.jetstack.io
helm repo add prometheus     https://prometheus-community.github.io/helm-charts
helm repo update

echo "Helm 安裝完成"
```

---

## 14. 安裝後驗證

### 14.1 完整驗證腳本

```bash
#!/bin/bash
# 在各節點上執行此腳本，驗證 K8s 前置配置是否完成
# 使用方式：bash verify-node.sh

PASS=0; FAIL=0

check() {
    local desc="$1"
    local cmd="$2"
    local expected="$3"

    result=$(eval "$cmd" 2>/dev/null)
    if [[ "$result" == *"$expected"* ]]; then
        echo "  PASS: $desc"
        ((PASS++))
    else
        echo "  FAIL: $desc（期望: '$expected'，實際: '$result'）"
        ((FAIL++))
    fi
}

check_zero() {
    local desc="$1"
    local cmd="$2"

    result=$(eval "$cmd" 2>/dev/null)
    if [[ -z "$result" ]]; then
        echo "  PASS: $desc（無輸出，正確）"
        ((PASS++))
    else
        echo "  FAIL: $desc（應無輸出，但有：'$result'）"
        ((FAIL++))
    fi
}

echo "=============================================="
echo "  K8s 節點前置配置驗證"
echo "  主機：$(hostname)"
echo "  時間：$(date '+%Y-%m-%d %H:%M:%S')"
echo "=============================================="

echo ""
echo "--- 系統基礎 ---"
check "Ubuntu 24.04" "lsb_release -rs" "24.04"

echo ""
echo "--- 時間同步 ---"
check "chrony 已啟動" "systemctl is-active chrony" "active"
check "時區設定正確" "timedatectl | grep 'Time zone'" "Asia/Taipei"

echo ""
echo "--- Swap ---"
check_zero "Swap 已關閉" "swapon --show"

echo ""
echo "--- Kernel 模組 ---"
check "overlay 已載入"       "lsmod | grep '^overlay'"       "overlay"
check "br_netfilter 已載入"   "lsmod | grep br_netfilter"     "br_netfilter"
check "ip_vs 已載入"          "lsmod | grep '^ip_vs '"        "ip_vs"

echo ""
echo "--- Sysctl 參數 ---"
check "bridge-nf-call-iptables=1" "sysctl -n net.bridge.bridge-nf-call-iptables" "1"
check "ip_forward=1"               "sysctl -n net.ipv4.ip_forward"                "1"
check "vm.swappiness=0"            "sysctl -n vm.swappiness"                      "0"
check "inotify.max_user_watches"   "sysctl -n fs.inotify.max_user_watches"        "1048576"

echo ""
echo "--- containerd ---"
check "containerd 已安裝"  "containerd --version"           "containerd"
check "containerd 已啟動"  "systemctl is-active containerd" "active"
check "SystemdCgroup 啟用" "grep SystemdCgroup /etc/containerd/config.toml" "true"

echo ""
echo "--- K8s 工具 ---"
check "kubeadm 已安裝"     "kubeadm version 2>&1"     "v1.30"
check "kubectl 已安裝"     "kubectl version --client 2>&1" "v1.30"
check "kubelet 已安裝"     "kubelet --version"         "v1.30"
check "kubelet 自動啟動"   "systemctl is-enabled kubelet" "enabled"

echo ""
echo "--- 防火牆 ---"
check "UFW 已啟用" "ufw status | head -1" "Status: active"

echo ""
echo "=============================================="
echo "  結果：PASS=$PASS  FAIL=$FAIL"
if [[ $FAIL -eq 0 ]]; then
    echo "  所有檢查通過！此節點已準備好進行 K8s 安裝。"
else
    echo "  有 $FAIL 項檢查失敗，請修復後再繼續。"
fi
echo "=============================================="
```

### 14.2 批次執行驗證

```bash
# ── [管理機] 使用 Ansible 在所有節點執行驗證 ─────────────────

# 將驗證腳本儲存
cat > ~/k8s-ansible/verify-node.sh << 'SCRIPT'
# （貼上 14.1 的完整腳本）
SCRIPT

# 在所有 K8s 節點執行
for ip in 192.168.20.11 192.168.20.12 192.168.20.13 \
          192.168.30.11 192.168.30.12 192.168.30.13; do
    echo ""
    echo "=== 驗證節點 $ip ==="
    ssh -i ~/.ssh/k8s_admin_key ubuntu@$ip 'bash -s' < ~/k8s-ansible/verify-node.sh
done
```

---

> **下一步**：完成所有節點的基礎配置並通過驗證後，請繼續閱讀
> `03-k8s-ha-cluster-install.md`，開始建立 Kubernetes HA 叢集。
>
> **重要確認清單**：
> - 所有節點的 `verify-node.sh` 輸出均為 PASS
> - 所有節點的 hostname 已正確設定
> - 所有節點的 `/etc/hosts` 已包含叢集全部節點資訊
> - 管理機可以 SSH 免密碼連線到所有節點
> - 所有節點時間同步誤差在 1 秒以內
