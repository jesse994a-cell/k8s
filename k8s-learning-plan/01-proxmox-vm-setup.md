# Proxmox VE 環境建置與 VM 建立

> **適用版本**：Proxmox VE 8.x
> **目標**：在 Proxmox 上建立 10 台 VM（3 master + 3 worker + 3 ceph + 1 gitlab）
> **撰寫日期**：2026-04-29

---

## 目錄

1. [Proxmox VE 安裝](#1-proxmox-ve-安裝)
2. [Proxmox 網路設定](#2-proxmox-網路設定)
3. [Ubuntu 24.04 Cloud Image 準備](#3-ubuntu-2404-cloud-image-準備)
4. [VM 建立詳細步驟](#4-vm-建立詳細步驟)
5. [Cloud-init 配置](#5-cloud-init-配置)
6. [批次建立 VM 腳本](#6-批次建立-vm-腳本)
7. [Ceph Node 額外磁碟新增](#7-ceph-node-額外磁碟新增)
8. [VM 啟動後基礎驗證](#8-vm-啟動後基礎驗證)
9. [網路拓樸示意](#9-網路拓樸示意)

---

## 1. Proxmox VE 安裝

### 1.1 下載 Proxmox VE ISO

```bash
# 從官方網站下載最新版 ISO
# https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso
# 目前版本：Proxmox VE 8.x（基於 Debian 12 Bookworm）
```

### 1.2 製作開機 USB

```bash
# 在 Linux/macOS 上使用 dd 指令
# 注意：/dev/sdX 請替換為實際 USB 裝置路徑（使用 lsblk 確認）
sudo dd if=proxmox-ve_8.x-x.iso of=/dev/sdX bs=4M status=progress conv=fdatasync

# Windows 使用者可改用 Rufus 或 Etcher（GUI 工具）
```

### 1.3 安裝步驟說明

**硬碟分割建議（ZFS RAID-1）**

```
建議使用兩顆 SSD/NVMe 做 ZFS mirror：
  Target Harddisk: 選擇 zfs（RAID1）
  硬碟1: /dev/sda（或 nvme0n1）
  硬碟2: /dev/sdb（或 nvme1n1）

ZFS 進階設定：
  ashift:    12（512B sector）或 13（4KB sector，NVMe 建議）
  compress:  on（lz4 壓縮，效能影響極小）
  checksum:  on（資料完整性驗證）

注意：若使用單顆磁碟，選 ext4 即可
```

**國家與語言設定**

```
Country:        Taiwan
Time zone:      Asia/Taipei
Keyboard Layout: U.S. English
```

**管理網路設定**

```
Management Interface: 選擇連接管理網路的 NIC
Hostname (FQDN):      pve01.yourdomain.com
IP Address:           192.168.10.11/24
Gateway:              192.168.10.1
DNS Server:           192.168.10.1
```

### 1.4 安裝後基礎設定

```bash
# SSH 連線到 Proxmox
ssh root@192.168.10.11

# 停用企業版 repo（無訂閱時）
echo "# deb https://enterprise.proxmox.com/debian/pve bookworm pve-enterprise" \
  > /etc/apt/sources.list.d/pve-enterprise.list

# 啟用社群版 repo
cat > /etc/apt/sources.list.d/pve-no-subscription.list << 'EOF'
deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription
EOF

# 更新系統
apt update && apt upgrade -y

# 安裝常用工具
apt install -y vim curl wget git htop iotop

# 移除每次登入的「無訂閱」彈窗提示（可選）
sed -i.backup -e "s/data.status.toLowerCase() !== 'active'/false/g" \
  /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
systemctl restart pveproxy.service
```

---

## 2. Proxmox 網路設定

### 2.1 網路架構規劃

```
物理主機 NIC 規劃：
  enp1s0  → 管理網路 bridge（vmbr0，VLAN 10）
  enp2s0  → VM 業務網路 bridge（vmbr1，VLAN trunk）
  enp3s0  → Ceph storage 專用（選配）
```

### 2.2 設定 /etc/network/interfaces

```bash
# 備份原始設定
cp /etc/network/interfaces /etc/network/interfaces.bak

cat > /etc/network/interfaces << 'EOF'
auto lo
iface lo inet loopback

# 管理網路 NIC
auto enp1s0
iface enp1s0 inet manual

# 業務網路 NIC（VM 流量，trunk port）
auto enp2s0
iface enp2s0 inet manual

# 管理網路 Bridge（Proxmox Web UI）
auto vmbr0
iface vmbr0 inet static
    address 192.168.10.11/24
    gateway 192.168.10.1
    bridge-ports enp1s0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware no
    dns-nameservers 192.168.10.1 8.8.8.8

# VM 業務網路 Bridge（支援 VLAN aware）
auto vmbr1
iface vmbr1 inet manual
    bridge-ports enp2s0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094
    # VLAN 20=K8s Master, 30=Worker, 40=Ceph, 50=GitLab
EOF

# 套用網路設定
ifreload -a
```

### 2.3 驗證網路設定

```bash
# 確認 bridge 狀態
brctl show

# 確認 IP 位址
ip addr show

# 測試連線
ping -c 4 192.168.10.1

# 查看 bridge VLAN 設定
bridge vlan show
```

---

## 3. Ubuntu 24.04 Cloud Image 準備

### 3.1 下載 Cloud Image

```bash
# 在 Proxmox 主機上執行
cd /var/lib/vz/template/iso/

# 下載 Ubuntu 24.04 minimal cloud image（qcow2 格式）
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img

# 確認檔案完整性
wget https://cloud-images.ubuntu.com/noble/current/SHA256SUMS
grep noble-server-cloudimg-amd64.img SHA256SUMS | sha256sum -c

# 查看映像資訊
qemu-img info noble-server-cloudimg-amd64.img
```

### 3.2 安裝必要套件

```bash
# 安裝 libguestfs-tools（用於修改 cloud image）
apt install -y libguestfs-tools

# 確認 virt-customize 可用
virt-customize --version
```

---

## 4. VM 建立詳細步驟

### 4.1 手動建立 VM（以 k8s-master1 為例）

#### 步驟 1：建立 VM 基礎結構

```bash
# 建立 VM（VM ID: 201，名稱: k8s-master1）
qm create 201 \
  --name k8s-master1 \
  --memory 8192 \
  --balloon 0 \
  --cores 4 \
  --sockets 1 \
  --cpu cputype=host \
  --net0 virtio,bridge=vmbr1,tag=20 \
  --ostype l26 \
  --bios ovmf \
  --machine q35 \
  --efidisk0 local-lvm:1,efitype=4m,pre-enrolled-keys=0 \
  --scsihw virtio-scsi-pci \
  --serial0 socket \
  --vga serial0 \
  --onboot 1

# 參數說明：
# --memory 8192        → 8GB RAM
# --balloon 0          → 停用記憶體 balloon（K8s 不建議使用）
# --cores 4            → 4 個 vCPU
# --cpu cputype=host   → 傳遞 host CPU 特性（效能最佳）
# --net0 virtio,bridge=vmbr1,tag=20 → 使用 VLAN 20
# --bios ovmf          → 使用 UEFI
# --serial0 socket     → 啟用串列控制台（cloud-init 需要）
```

#### 步驟 2：匯入磁碟並調整大小

```bash
# 將 cloud image 匯入為 VM 磁碟
qm importdisk 201 /var/lib/vz/template/iso/noble-server-cloudimg-amd64.img local-lvm

# 掛載磁碟到 VM
qm set 201 --scsi0 local-lvm:vm-201-disk-1,discard=on,ssd=1,cache=writeback

# 調整磁碟大小到 80GB
qm resize 201 scsi0 80G
```

#### 步驟 3：新增 cloud-init 磁碟

```bash
# 新增 cloud-init 磁碟
qm set 201 --ide2 local-lvm:cloudinit

# 設定開機順序（從 scsi0 開機）
qm set 201 --boot order=scsi0
```

#### 步驟 4：設定 cloud-init 參數

```bash
# 生成 SSH key（若尚未生成）
ssh-keygen -t ed25519 -C "k8s-cluster-admin" -f /root/.ssh/k8s_admin_key -N ""

# 設定 cloud-init 使用者
qm set 201 --ciuser ubuntu

# 設定 SSH 公鑰
qm set 201 --sshkeys /root/.ssh/k8s_admin_key.pub

# 設定靜態 IP（VLAN 20 網段）
qm set 201 --ipconfig0 ip=192.168.20.11/24,gw=192.168.20.1

# 設定 DNS
qm set 201 --nameserver "8.8.8.8 8.8.4.4"
qm set 201 --searchdomain yourdomain.com
```

#### 步驟 5：啟動 VM

```bash
# 啟動 VM
qm start 201

# 查看啟動狀態
qm status 201

# 查看 console（等待 cloud-init 完成）
# 在 Proxmox Web UI 點選 VM > Console
```

### 4.2 所有 VM 規格總表

| VM ID | Hostname | vCPU | RAM | 磁碟 | VLAN | IP |
|-------|----------|------|-----|------|------|-----|
| 201 | k8s-master1 | 4 | 8GB | 80GB | 20 | 192.168.20.11 |
| 202 | k8s-master2 | 4 | 8GB | 80GB | 20 | 192.168.20.12 |
| 203 | k8s-master3 | 4 | 8GB | 80GB | 20 | 192.168.20.13 |
| 211 | k8s-worker1 | 8 | 16GB | 100GB | 30 | 192.168.30.11 |
| 212 | k8s-worker2 | 8 | 16GB | 100GB | 30 | 192.168.30.12 |
| 213 | k8s-worker3 | 8 | 16GB | 100GB | 30 | 192.168.30.13 |
| 221 | ceph-node1 | 4 | 8GB | 50GB + 200GB x3 | 40 | 192.168.40.11 |
| 222 | ceph-node2 | 4 | 8GB | 50GB + 200GB x3 | 40 | 192.168.40.12 |
| 223 | ceph-node3 | 4 | 8GB | 50GB + 200GB x3 | 40 | 192.168.40.13 |
| 231 | gitlab-server | 8 | 16GB | 200GB | 50 | 192.168.50.11 |

---

## 5. Cloud-init 配置

### 5.1 建立 cloud-init snippets 目錄

```bash
# 建立 snippets 目錄（Proxmox 需要此目錄存放自訂 cloud-init 設定）
mkdir -p /var/lib/vz/snippets

# 確認 Proxmox storage 設定包含 snippets
# 在 Proxmox Web UI: Datacenter > Storage > local > 勾選 Snippets
# 或使用 CLI：
pvesm set local --content vztmpl,iso,backup,snippets
```

### 5.2 通用 user-data 設定

```bash
SSH_KEY=$(cat /root/.ssh/k8s_admin_key.pub)

cat > /var/lib/vz/snippets/common-user-data.yaml << EOF
#cloud-config

users:
  - name: ubuntu
    gecos: K8s Admin
    groups: sudo
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    ssh_authorized_keys:
      - ${SSH_KEY}

disable_root: true
ssh_pwauth: false

package_update: true
packages:
  - vim
  - curl
  - wget
  - git
  - htop
  - net-tools
  - bash-completion
  - jq

timezone: Asia/Taipei

runcmd:
  - swapoff -a
  - sed -i '/\bswap\b/d' /etc/fstab
  - echo "overlay" > /etc/modules-load.d/k8s.conf
  - echo "br_netfilter" >> /etc/modules-load.d/k8s.conf
  - modprobe overlay
  - modprobe br_netfilter
  - echo "cloud-init completed: $(hostname)" > /tmp/cloud-init-done

final_message: "Cloud-init 初始化完成 - 版本: \$version"
EOF

echo "通用 user-data 設定檔建立完成"
```

### 5.3 為各節點建立 network-data

```bash
# 函數：建立 network-data
create_network_config() {
    local hostname="$1"
    local ip="$2"
    local gateway="$3"

    cat > "/var/lib/vz/snippets/${hostname}-network.yaml" << EOF
version: 2
ethernets:
  eth0:
    dhcp4: false
    addresses:
      - ${ip}
    routes:
      - to: default
        via: ${gateway}
    nameservers:
      addresses:
        - 8.8.8.8
        - 8.8.4.4
      search:
        - cluster.local
        - yourdomain.com
EOF
}

# 建立各節點網路設定
create_network_config "k8s-master1" "192.168.20.11/24" "192.168.20.1"
create_network_config "k8s-master2" "192.168.20.12/24" "192.168.20.1"
create_network_config "k8s-master3" "192.168.20.13/24" "192.168.20.1"
create_network_config "k8s-worker1" "192.168.30.11/24" "192.168.30.1"
create_network_config "k8s-worker2" "192.168.30.12/24" "192.168.30.1"
create_network_config "k8s-worker3" "192.168.30.13/24" "192.168.30.1"
create_network_config "ceph-node1"  "192.168.40.11/24" "192.168.40.1"
create_network_config "ceph-node2"  "192.168.40.12/24" "192.168.40.1"
create_network_config "ceph-node3"  "192.168.40.13/24" "192.168.40.1"
create_network_config "gitlab-server" "192.168.50.11/24" "192.168.50.1"

echo "所有節點 network-data 建立完成"
ls -la /var/lib/vz/snippets/
```

### 5.4 套用 cicustom 到 VM

```bash
# 套用自訂 cloud-init 設定
for vmid_host in "201:k8s-master1" "202:k8s-master2" "203:k8s-master3" \
                 "211:k8s-worker1" "212:k8s-worker2" "213:k8s-worker3" \
                 "221:ceph-node1"  "222:ceph-node2"  "223:ceph-node3" \
                 "231:gitlab-server"; do
    vmid="${vmid_host%%:*}"
    hostname="${vmid_host##*:}"

    qm set "$vmid" --cicustom \
      "user=local:snippets/common-user-data.yaml,network=local:snippets/${hostname}-network.yaml"

    echo "VM $vmid ($hostname) cicustom 套用完成"
done
```

### 5.5 驗證 cloud-init 設定

```bash
# 查看 VM 的 cloud-init 設定（以 VM 201 為例）
qm cloudinit dump 201 user
qm cloudinit dump 201 network
```

---

## 6. 批次建立 VM 腳本

```bash
#!/bin/bash
# ============================================================
# Proxmox 批次建立 K8s VM 腳本
# 使用方式：bash /root/scripts/create-k8s-vms.sh
# ============================================================

set -euo pipefail

# ── 全域變數 ────────────────────────────────────────────────
CLOUD_IMAGE="/var/lib/vz/template/iso/noble-server-cloudimg-amd64.img"
STORAGE="local-lvm"
SNIPPETS_DIR="/var/lib/vz/snippets"
SSH_KEY_FILE="/root/.ssh/k8s_admin_key.pub"
BRIDGE="vmbr1"
DNS_SERVERS="8.8.8.8 8.8.4.4"

GREEN='\033[0;32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m'

log()   { echo -e "${GREEN}[INFO]${NC} $1"; }
warn()  { echo -e "${YELLOW}[WARN]${NC} $1"; }
error() { echo -e "${RED}[ERROR]${NC} $1"; exit 1; }

# ── 前置檢查 ────────────────────────────────────────────────
check_prerequisites() {
    log "檢查前置條件..."
    [[ -f "$CLOUD_IMAGE" ]]  || error "找不到 Cloud Image：$CLOUD_IMAGE"
    [[ -f "$SSH_KEY_FILE" ]] || error "找不到 SSH 公鑰：$SSH_KEY_FILE"
    pvesm status | grep -q "$STORAGE" || error "找不到 storage：$STORAGE"
    log "前置條件檢查通過"
}

# ── 建立 cloud-init snippets ─────────────────────────────────
create_snippets() {
    mkdir -p "$SNIPPETS_DIR"
    local ssh_key
    ssh_key=$(cat "$SSH_KEY_FILE")

    cat > "${SNIPPETS_DIR}/common-user-data.yaml" << YAML
#cloud-config
users:
  - name: ubuntu
    groups: sudo
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    ssh_authorized_keys:
      - ${ssh_key}
disable_root: true
ssh_pwauth: false
package_update: true
packages:
  - vim
  - curl
  - wget
  - git
  - htop
  - net-tools
timezone: Asia/Taipei
runcmd:
  - swapoff -a
  - sed -i '/\\bswap\\b/d' /etc/fstab
  - echo overlay > /etc/modules-load.d/k8s.conf
  - echo br_netfilter >> /etc/modules-load.d/k8s.conf
  - modprobe overlay
  - modprobe br_netfilter
YAML
    log "Cloud-init snippets 建立完成"
}

# ── 建立單一 VM ──────────────────────────────────────────────
# 參數：$1=VMID $2=hostname $3=cores $4=memory $5=disk $6=vlan $7=ip $8=gw
create_vm() {
    local vmid="$1" hostname="$2" cores="$3" memory="$4"
    local disk_size="$5" vlan="$6" ip="$7" gateway="$8"

    log "建立 VM $vmid ($hostname)..."

    if qm status "$vmid" &>/dev/null; then
        warn "VM $vmid 已存在，跳過..."
        return 0
    fi

    # 建立 VM
    qm create "$vmid" \
        --name "$hostname" \
        --memory "$memory" \
        --balloon 0 \
        --cores "$cores" \
        --sockets 1 \
        --cpu cputype=host \
        --net0 "virtio,bridge=${BRIDGE},tag=${vlan}" \
        --ostype l26 \
        --bios ovmf \
        --machine q35 \
        --efidisk0 "${STORAGE}:1,efitype=4m,pre-enrolled-keys=0" \
        --scsihw virtio-scsi-pci \
        --serial0 socket \
        --vga serial0 \
        --onboot 1

    # 匯入並掛載磁碟
    qm importdisk "$vmid" "$CLOUD_IMAGE" "$STORAGE" --format qcow2
    qm set "$vmid" --scsi0 "${STORAGE}:vm-${vmid}-disk-1,discard=on,ssd=1,cache=writeback"
    qm resize "$vmid" scsi0 "$disk_size"

    # 新增 cloud-init 磁碟
    qm set "$vmid" --ide2 "${STORAGE}:cloudinit"
    qm set "$vmid" --boot order=scsi0

    # 建立節點網路設定
    cat > "${SNIPPETS_DIR}/${hostname}-network.yaml" << YAML
version: 2
ethernets:
  eth0:
    dhcp4: false
    addresses:
      - ${ip}
    routes:
      - to: default
        via: ${gateway}
    nameservers:
      addresses:
        - 8.8.8.8
        - 8.8.4.4
YAML

    # 套用 cloud-init
    qm set "$vmid" \
        --ciuser ubuntu \
        --sshkeys "$SSH_KEY_FILE" \
        --nameserver "$DNS_SERVERS" \
        --cicustom "user=local:snippets/common-user-data.yaml,network=local:snippets/${hostname}-network.yaml"

    log "VM $vmid ($hostname) 建立完成"
}

# ── 新增 Ceph OSD 磁碟 ────────────────────────────────────────
add_ceph_disks() {
    local vmid="$1"
    log "為 VM $vmid 新增 Ceph OSD 磁碟（3 x 200GB）..."
    for i in 1 2 3; do
        qm set "$vmid" --scsi${i} "${STORAGE}:200,discard=on,ssd=1,cache=writeback"
    done
    log "VM $vmid Ceph 磁碟新增完成"
}

# ── 主流程 ────────────────────────────────────────────────────
main() {
    log "=== 開始批次建立 K8s 叢集 VM ==="

    check_prerequisites
    create_snippets

    # Master Nodes
    log "--- 建立 Master Nodes ---"
    create_vm 201 "k8s-master1" 4 8192  "80G"  20 "192.168.20.11/24" "192.168.20.1"
    create_vm 202 "k8s-master2" 4 8192  "80G"  20 "192.168.20.12/24" "192.168.20.1"
    create_vm 203 "k8s-master3" 4 8192  "80G"  20 "192.168.20.13/24" "192.168.20.1"

    # Worker Nodes
    log "--- 建立 Worker Nodes ---"
    create_vm 211 "k8s-worker1" 8 16384 "100G" 30 "192.168.30.11/24" "192.168.30.1"
    create_vm 212 "k8s-worker2" 8 16384 "100G" 30 "192.168.30.12/24" "192.168.30.1"
    create_vm 213 "k8s-worker3" 8 16384 "100G" 30 "192.168.30.13/24" "192.168.30.1"

    # Ceph Nodes
    log "--- 建立 Ceph Storage Nodes ---"
    create_vm 221 "ceph-node1"  4 8192  "50G"  40 "192.168.40.11/24" "192.168.40.1"
    create_vm 222 "ceph-node2"  4 8192  "50G"  40 "192.168.40.12/24" "192.168.40.1"
    create_vm 223 "ceph-node3"  4 8192  "50G"  40 "192.168.40.13/24" "192.168.40.1"

    # 為 Ceph nodes 新增資料磁碟
    add_ceph_disks 221
    add_ceph_disks 222
    add_ceph_disks 223

    # GitLab Server
    log "--- 建立 GitLab Server ---"
    create_vm 231 "gitlab-server" 8 16384 "200G" 50 "192.168.50.11/24" "192.168.50.1"

    # 啟動所有 VM（間隔 3 秒，避免 I/O 衝擊）
    log "--- 啟動所有 VM ---"
    for vmid in 201 202 203 211 212 213 221 222 223 231; do
        log "啟動 VM $vmid..."
        qm start "$vmid"
        sleep 3
    done

    log "=== 所有 VM 建立並啟動完成 ==="
    log "等待 90 秒讓 cloud-init 完成初始化..."
    sleep 90

    log "=== VM 狀態總覽 ==="
    qm list
}

main "$@"
```

### 6.1 執行腳本

```bash
# 建立腳本目錄
mkdir -p /root/scripts

# 將上方腳本儲存到檔案
vim /root/scripts/create-k8s-vms.sh

# 賦予執行權限
chmod +x /root/scripts/create-k8s-vms.sh

# 先生成 SSH key（若尚未生成）
ssh-keygen -t ed25519 -C "k8s-cluster-admin" -f /root/.ssh/k8s_admin_key -N ""

# 執行腳本
bash /root/scripts/create-k8s-vms.sh 2>&1 | tee /root/scripts/create-vms.log
```

---

## 7. Ceph Node 額外磁碟新增

### 7.1 透過 CLI 新增磁碟

```bash
# 為 ceph-node1 (VM 221) 新增 3 顆 200GB OSD 磁碟
# scsi1、scsi2、scsi3 為資料磁碟（scsi0 是系統磁碟）
qm set 221 --scsi1 "local-lvm:200,discard=on,ssd=1"
qm set 221 --scsi2 "local-lvm:200,discard=on,ssd=1"
qm set 221 --scsi3 "local-lvm:200,discard=on,ssd=1"

# 同樣操作套用到 ceph-node2 (222) 和 ceph-node3 (223)
for vmid in 222 223; do
    qm set $vmid --scsi1 "local-lvm:200,discard=on,ssd=1"
    qm set $vmid --scsi2 "local-lvm:200,discard=on,ssd=1"
    qm set $vmid --scsi3 "local-lvm:200,discard=on,ssd=1"
done

# 確認磁碟設定
qm config 221 | grep scsi
```

### 7.2 在 Ceph VM 內驗證磁碟

```bash
# SSH 進入 ceph-node1
ssh -i /root/.ssh/k8s_admin_key ubuntu@192.168.40.11

# 確認磁碟被識別
lsblk
# 預期輸出：
# NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
# sda      8:0   0   50G  0  disk        ← 系統磁碟
# ├─sda1   8:1   0   50G  0  part /
# sdb      8:16  0  200G  0  disk        ← Ceph OSD 磁碟 1
# sdc      8:32  0  200G  0  disk        ← Ceph OSD 磁碟 2
# sdd      8:48  0  200G  0  disk        ← Ceph OSD 磁碟 3

# 確認磁碟無分割表（Ceph OSD 需要空白磁碟）
lsblk -f /dev/sdb /dev/sdc /dev/sdd
# FSTYPE 欄位應為空
```

---

## 8. VM 啟動後基礎驗證

### 8.1 批次連線驗證腳本

```bash
#!/bin/bash
# 在 Proxmox 主機上執行，驗證所有 VM 連線狀態

SSH_KEY="/root/.ssh/k8s_admin_key"
SSH_OPTS="-o StrictHostKeyChecking=no -o ConnectTimeout=10 -i $SSH_KEY"

declare -A VMS=(
    ["k8s-master1"]="192.168.20.11"
    ["k8s-master2"]="192.168.20.12"
    ["k8s-master3"]="192.168.20.13"
    ["k8s-worker1"]="192.168.30.11"
    ["k8s-worker2"]="192.168.30.12"
    ["k8s-worker3"]="192.168.30.13"
    ["ceph-node1"]="192.168.40.11"
    ["ceph-node2"]="192.168.40.12"
    ["ceph-node3"]="192.168.40.13"
    ["gitlab-server"]="192.168.50.11"
)

echo "============================================"
echo " VM 連線驗證報告"
echo "============================================"
printf "%-20s %-18s %-10s %-15s\n" "Hostname" "IP" "SSH" "OS版本"
echo "--------------------------------------------"

PASS=0; FAIL=0

for hostname in $(echo "${!VMS[@]}" | tr ' ' '\n' | sort); do
    ip="${VMS[$hostname]}"
    if ssh $SSH_OPTS ubuntu@"$ip" "echo ok" &>/dev/null; then
        os_ver=$(ssh $SSH_OPTS ubuntu@"$ip" "lsb_release -rs" 2>/dev/null)
        printf "%-20s %-18s %-10s %-15s\n" "$hostname" "$ip" "OK" "Ubuntu $os_ver"
        ((PASS++))
    else
        printf "%-20s %-18s %-10s %-15s\n" "$hostname" "$ip" "FAIL" "無法連線"
        ((FAIL++))
    fi
done

echo "============================================"
echo " 結果：PASS=$PASS  FAIL=$FAIL"
echo "============================================"
```

### 8.2 個別節點手動驗證

```bash
# SSH 連線測試（以 master1 為例）
ssh -i /root/.ssh/k8s_admin_key ubuntu@192.168.20.11

# 在 VM 內執行基礎檢查
hostname                   # 應為 k8s-master1
ip addr show eth0          # 應顯示 192.168.20.11/24
ping -c 2 8.8.8.8          # 外網連線測試
ping -c 2 192.168.20.12    # 節點間連線測試
df -h /                    # 確認磁碟大小（應為 ~80GB）
free -h                    # 確認記憶體（應為 ~8GB）
nproc                      # 確認 CPU 數量（應為 4）
swapon --show              # 應無輸出（swap 已關閉）
```

---

## 9. 網路拓樸示意

```
                    [Internet / 上游路由器]
                              │
                   ┌──────────┴──────────┐
                   │   Managed Switch    │
                   │  (802.1Q VLAN Trunk)│
                   └──────────┬──────────┘
                              │
                   ┌──────────┴──────────┐
                   │   Proxmox VE Host   │
                   │   192.168.10.11     │
                   │                     │
                   │  vmbr0 (VLAN 10)    │  ← Proxmox 管理介面
                   │  vmbr1 (VLAN trunk) │  ← 所有 VM 流量
                   └──────────┬──────────┘
                              │
       ┌──────────────────────┼─────────────────────────┐
       │                      │                          │
  ┌────▼────────┐      ┌───────▼──────┐        ┌────────▼─────┐
  │  VLAN 20    │      │   VLAN 30    │        │   VLAN 40/41 │
  │ K8s Control │      │  K8s Worker  │        │ Ceph Storage │
  │             │      │              │        │              │
  │ master1 .11 │      │  worker1 .11 │        │ ceph1  .40.11│
  │ master2 .12 │      │  worker2 .12 │        │ ceph2  .40.12│
  │ master3 .13 │      │  worker3 .13 │        │ ceph3  .40.13│
  │ VIP    .100 │      │              │        │ cluster:41.x │
  └─────────────┘      └──────────────┘        └──────────────┘

       ┌──────────────────────────────┐
       │          VLAN 50             │
       │         CI/CD 網路           │
       │                              │
       │  gitlab-server  .50.11       │
       │  ArgoCD (K8s 內部 Service)   │
       └──────────────────────────────┘

  Pod Network（CNI 虛擬）: 10.244.0.0/16
  Service Network（虛擬）: 10.96.0.0/12
```

### 9.1 各 VLAN 通訊矩陣

```
         VLAN10  VLAN20  VLAN30  VLAN40  VLAN50
VLAN10    ✓       ✓       ✓       ✓       ✓     ← 管理網路（全通）
VLAN20    ✓       ✓       ✓       ✓       ✓     ← K8s Master
VLAN30    ✗       ✓       ✓       ✓       ✗     ← Worker（存取 Master+Ceph）
VLAN40    ✗       ✓       ✓       ✓       ✗     ← Ceph（存取 K8s）
VLAN50    ✗       ✓       ✓       ✗       ✓     ← CI/CD（存取 K8s）
```

---

> **下一步**：完成 VM 建立後，請繼續閱讀 [02-ubuntu24-base-config.md](02-ubuntu24-base-config.md)
> 進行 Ubuntu 系統初始化配置。
>
> **注意**：進行下一步前，請確認所有 VM 都能透過 SSH 正常連線，且 IP 位址設定正確。
