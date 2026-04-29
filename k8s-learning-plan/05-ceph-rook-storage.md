# 05 - CEPH 與 Rook 儲存系統完整指南

> 環境：CEPH node x3（ceph1: 192.168.100.31, ceph2: 192.168.100.32, ceph3: 192.168.100.33）
> 每台節點配備 3 顆 200GB 資料磁碟（/dev/sdb、/dev/sdc、/dev/sdd）
> K8s 叢集：3 master + 3 worker，Ubuntu 24.04

---

## 1. CEPH 基礎概念

### 1.1 CEPH 架構說明

CEPH 是一個分散式儲存系統，支援物件儲存、區塊儲存與檔案系統三種儲存模式。

#### 核心元件說明

| 元件 | 全名 | 職責 |
|------|------|------|
| **MON** | Monitor | 維護叢集狀態 Map（叢集拓撲、OSD 狀態）。需要奇數個（3 或 5）以達成 quorum |
| **OSD** | Object Storage Daemon | 實際儲存資料的程序，每顆磁碟對應一個 OSD。負責資料複製、恢復、平衡 |
| **MGR** | Manager | 提供 Dashboard UI、監控指標（Prometheus）、模組管理 |
| **MDS** | Metadata Server | 僅 CephFS 需要，負責管理檔案系統的 metadata |
| **RGW** | RADOS Gateway | 提供 S3/Swift 相容的物件儲存 HTTP API |
| **CRUSH** | - | CEPH 的資料分佈演算法，決定資料放在哪些 OSD |

#### 儲存類型

- **RBD (RADOS Block Device)**：區塊儲存，對應 K8s 的 ReadWriteOnce PVC
- **CephFS**：POSIX 相容分散式檔案系統，對應 K8s 的 ReadWriteMany PVC
- **RGW Object Storage**：S3/Swift 相容，適合非結構化資料

### 1.2 Rook-CEPH 是什麼？

Rook 是一個 Kubernetes Operator，專門用來在 K8s 叢集「內部」部署和管理 CEPH。

**使用時機比較：**

| 方案 | 適用場景 | 優點 | 缺點 |
|------|----------|------|------|
| **獨立 CEPH + CSI** | 生產環境、需要獨立維護 CEPH | 靈活、可獨立升級、節點隔離 | 需額外 VM 資源 |
| **Rook-CEPH** | 全 K8s 管理、測試環境 | 統一管理、GitOps 友好 | CEPH 與 K8s 生命週期耦合 |

**本指南主要採用：獨立 CEPH 叢集 + K8s CSI 串接（生產推薦方案）**

### 1.3 架構圖（ASCII）

```
┌─────────────────────────────────────────────────────────────────┐
│                        CEPH 叢集                                 │
│                                                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   ceph1      │  │   ceph2      │  │   ceph3      │             │
│  │ .100.31      │  │ .100.32      │  │ .100.33      │             │
│  │             │  │             │  │             │             │
│  │ MON  MGR   │  │ MON         │  │ MON         │             │
│  │ MDS  RGW   │  │ MDS         │  │             │             │
│  │             │  │             │  │             │             │
│  │ OSD0(sdb)  │  │ OSD3(sdb)  │  │ OSD6(sdb)  │             │
│  │ OSD1(sdc)  │  │ OSD4(sdc)  │  │ OSD7(sdc)  │             │
│  │ OSD2(sdd)  │  │ OSD5(sdd)  │  │ OSD8(sdd)  │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
└─────────────────────────────────────────────────────────────────┘
                              │
                    CSI Driver（rbd/cephfs）
                              │
┌─────────────────────────────────────────────────────────────────┐
│                      Kubernetes 叢集                              │
│                                                                   │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────────┐    │
│  │  master-1  │  │  master-2  │  │       master-3          │    │
│  └────────────┘  └────────────┘  └────────────────────────┘    │
│                                                                   │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────────┐    │
│  │  worker-1  │  │  worker-2  │  │       worker-3          │    │
│  │            │  │            │  │                          │    │
│  │ ceph-csi   │  │ ceph-csi   │  │       ceph-csi           │    │
│  │ node plugin│  │ node plugin│  │       node plugin        │    │
│  └────────────┘  └────────────┘  └────────────────────────┘    │
│                                                                   │
│  StorageClass: ceph-rbd   StorageClass: ceph-cephfs             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 獨立 CEPH 叢集建置（使用 cephadm）

### 2.1 前置準備（所有 CEPH 節點）

```bash
# 所有節點執行（ceph1、ceph2、ceph3）

# 更新系統
apt update && apt upgrade -y

# 安裝必要套件
apt install -y python3 python3-pip curl wget chronyd lvm2

# 設定時間同步（CEPH 對時間一致性要求嚴格）
systemctl enable --now chrony
chronyc tracking

# 設定 /etc/hosts（所有節點）
cat >> /etc/hosts << 'EOF'
192.168.100.31 ceph1
192.168.100.32 ceph2
192.168.100.33 ceph3
EOF

# 關閉 swap（建議）
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab

# 確認磁碟狀態（每台應看到 sdb, sdc, sdd）
lsblk
# 確認磁碟沒有任何 partition 或檔案系統
wipefs -a /dev/sdb /dev/sdc /dev/sdd  # 若有舊資料則清除
```

### 2.2 安裝 cephadm（僅在 ceph1 執行）

```bash
# 在 ceph1 執行

# 方法一：使用官方腳本安裝最新穩定版
curl --silent --remote-name --location https://github.com/ceph/ceph/raw/reef/src/cephadm/cephadm

chmod +x cephadm
mv cephadm /usr/local/bin/

# 確認版本
cephadm version

# 添加 CEPH Reef 官方套件來源（用於後續 ceph-common 安裝）
cephadm add-repo --release reef

# 安裝 ceph-common（提供 ceph CLI 指令）
cephadm install ceph-common
```

### 2.3 Bootstrap 第一個節點（ceph1）

```bash
# 在 ceph1 執行

# Bootstrap CEPH 叢集
# --mon-ip：本節點的 IP（用於 Monitor）
# --cluster-network：CEPH 內部複製流量網段（若有獨立網路則指定）
cephadm bootstrap \
  --mon-ip 192.168.100.31 \
  --cluster-network 192.168.100.0/24 \
  --initial-dashboard-user admin \
  --initial-dashboard-password "YourDashboardPass123!" \
  --allow-overwrite

# Bootstrap 完成後會顯示：
# Ceph Dashboard is now available at:
#   URL: https://ceph1:8443/
#   User: admin
#   Password: YourDashboardPass123!

# 確認叢集狀態
ceph status
ceph -s

# 預期輸出（初始狀態）：
#   cluster:
#     id:     xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
#     health: HEALTH_WARN  (初始警告，加入更多節點後會消失)
#   services:
#     mon: 1 daemons, quorum ceph1
#     mgr: ceph1.xxxxx(active)
#     osd: 0 osds: 0 up, 0 in
```

### 2.4 加入其他節點（ceph2、ceph3）

```bash
# 在 ceph1 執行

# 複製 SSH key 到其他節點（讓 cephadm 可以 SSH 管理）
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph2
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph3

# 將 ceph2 加入叢集
ceph orch host add ceph2 192.168.100.32

# 將 ceph3 加入叢集
ceph orch host add ceph3 192.168.100.33

# 確認所有節點已加入
ceph orch host ls

# 輸出範例：
# HOST   ADDR            LABELS  STATUS
# ceph1  192.168.100.31  _admin
# ceph2  192.168.100.32
# ceph3  192.168.100.33

# 在所有節點部署 MON（預設已自動，確認 3 個 MON）
ceph orch apply mon 3

# 等待 MON 部署完成
watch ceph -s

# 部署 MGR（建議 2 個，一主一備）
ceph orch apply mgr 2

# 確認 MGR 狀態
ceph mgr stat
```

### 2.5 新增 OSD（指定磁碟）

```bash
# 在 ceph1 執行

# 方法一：讓 cephadm 自動探索並使用所有可用磁碟
ceph orch apply osd --all-available-devices

# 方法二：逐一指定磁碟（推薦，確保控制）
# 為每台節點的每顆磁碟建立 OSD
ceph orch daemon add osd ceph1:/dev/sdb
ceph orch daemon add osd ceph1:/dev/sdc
ceph orch daemon add osd ceph1:/dev/sdd

ceph orch daemon add osd ceph2:/dev/sdb
ceph orch daemon add osd ceph2:/dev/sdc
ceph orch daemon add osd ceph2:/dev/sdd

ceph orch daemon add osd ceph3:/dev/sdb
ceph orch daemon add osd ceph3:/dev/sdc
ceph orch daemon add osd ceph3:/dev/sdd

# 方法三：使用 DriveGroups spec（最佳實踐）
cat > /tmp/osd-spec.yaml << 'EOF'
service_type: osd
service_id: all-data-disks
placement:
  hosts:
    - ceph1
    - ceph2
    - ceph3
spec:
  data_devices:
    paths:
      - /dev/sdb
      - /dev/sdc
      - /dev/sdd
EOF

ceph orch apply -i /tmp/osd-spec.yaml

# 等待 OSD 全部 up
watch ceph osd tree

# 確認 9 個 OSD 都是 up 狀態
ceph osd stat
# 輸出：9 osds: 9 up (since Xs), 9 in (since Xs)

# 確認叢集健康
ceph health
# 目標：HEALTH_OK
```

### 2.6 建立 RBD Pool（供 K8s Block Storage 使用）

```bash
# 在 ceph1 執行

# 建立 RBD Pool
ceph osd pool create kubernetes-rbd 128

# 啟用 RBD application
ceph osd pool application enable kubernetes-rbd rbd

# 初始化 Pool
rbd pool init kubernetes-rbd

# 設定複製數（預設 3，與 OSD 節點數一致）
ceph osd pool set kubernetes-rbd size 3
ceph osd pool set kubernetes-rbd min_size 2

# 確認 Pool 設定
ceph osd pool get kubernetes-rbd all

# 建立 CSI 用的使用者（K8s CSI 會用此帳號操作 CEPH）
ceph auth get-or-create client.kubernetes-csi \
  mon 'profile rbd' \
  osd 'profile rbd pool=kubernetes-rbd' \
  mgr 'profile rbd pool=kubernetes-rbd' \
  -o /etc/ceph/ceph.client.kubernetes-csi.keyring

# 查看 key（後面 K8s Secret 會用到）
ceph auth get-key client.kubernetes-csi
# 輸出範例：AQCxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==
```

### 2.7 建立 CephFS（供 K8s Shared Storage 使用）

```bash
# 在 ceph1 執行

# 建立 CephFS（自動建立 metadata pool 和 data pool）
ceph fs volume create kubernetes-cephfs

# 確認 CephFS 狀態
ceph fs status

# 確認 MDS 已部署（CephFS 需要 MDS）
ceph orch ls mds
ceph mds stat

# 建立 subvolumegroup（K8s CSI 使用）
ceph fs subvolumegroup create kubernetes-cephfs csi

# 確認 subvolumegroup
ceph fs subvolumegroup ls kubernetes-cephfs

# 建立 CephFS CSI 用的使用者
ceph auth get-or-create client.cephfs-csi \
  mon 'allow r' \
  mds 'allow rw path=/' \
  osd 'allow rw tag cephfs *=kubernetes-cephfs' \
  mgr 'allow rw' \
  -o /etc/ceph/ceph.client.cephfs-csi.keyring

# 取得 admin key（建立 K8s Secret 時需要）
ceph auth get-key client.admin
# 輸出：AQCxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==

# 取得 FSID（叢集 ID）
ceph fsid
# 輸出：xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

### 2.8 CEPH 常用指令與狀態確認

```bash
# ===== 叢集狀態 =====
ceph status                          # 整體狀態摘要
ceph -s                              # 同上（簡寫）
ceph health                          # 健康狀態
ceph health detail                   # 健康狀態詳細資訊

# ===== OSD 管理 =====
ceph osd tree                        # OSD 拓撲樹狀圖
ceph osd stat                        # OSD 統計
ceph osd df                          # OSD 容量使用率
ceph osd df tree                     # 含拓撲的容量使用率
ceph osd perf                        # OSD 效能指標

# 暫停/恢復 OSD
ceph osd out osd.3                   # 將 OSD 3 標記為 out（開始資料遷移）
ceph osd in osd.3                    # 將 OSD 3 標記回 in

# ===== Pool 管理 =====
ceph osd pool ls                     # 列出所有 pool
ceph osd pool ls detail              # 列出 pool 詳細資訊
ceph df                              # 顯示叢集及 pool 容量
ceph osd pool stats                  # Pool 統計資料

# ===== MON 管理 =====
ceph mon stat                        # MON 統計
ceph mon dump                        # MON 設定

# ===== 效能監控 =====
ceph iostat                          # 即時 I/O 統計
ceph osd pool stats kubernetes-rbd  # 特定 pool 統計

# ===== RBD 管理 =====
rbd ls kubernetes-rbd                # 列出 pool 中的 image
rbd info kubernetes-rbd/test-image  # Image 詳細資訊
rbd du kubernetes-rbd               # 顯示實際使用容量

# ===== 事件與日誌 =====
ceph log last 20                     # 最近 20 條日誌
ceph -w                             # 即時監控叢集事件

# ===== orch 管理 =====
ceph orch ls                         # 列出所有 orchestrator 服務
ceph orch ps                         # 列出所有 daemon
ceph orch host ls                    # 列出所有主機
```

---

## 3. Rook-CEPH 方式（K8s 內部部署，備選方案）

> 注意：本節為備選方案說明。若使用獨立 CEPH，請直接跳至第 4 節。

### 3.1 安裝 Rook Operator

```bash
# 在 K8s master 節點執行

# 添加 Rook Helm repo
helm repo add rook-release https://charts.rook.io/release
helm repo update

# 建立 namespace
kubectl create namespace rook-ceph

# 安裝 Rook Operator
helm install --create-namespace \
  --namespace rook-ceph \
  rook-ceph rook-release/rook-ceph \
  --version v1.13.0 \
  --set csi.enableRbdDriver=true \
  --set csi.enableCephfsDriver=true

# 確認 Operator 狀態
kubectl -n rook-ceph get pods
kubectl -n rook-ceph get pod -l app=rook-ceph-operator
```

### 3.2 建立 CephCluster CR

```yaml
# ceph-cluster.yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v18.2.0
    allowUnsupported: false
  dataDirHostPath: /var/lib/rook
  skipUpgradeChecks: false
  continueUpgradeAfterChecksEvenIfNotHealthy: false
  mon:
    count: 3
    allowMultiplePerNode: false
  mgr:
    count: 2
    allowMultiplePerNode: false
    modules:
      - name: pg_autoscaler
        enabled: true
  dashboard:
    enabled: true
    ssl: true
  monitoring:
    enabled: false
  network:
    connections:
      encryption:
        enabled: false
      compression:
        enabled: false
  crashCollector:
    disable: false
  cleanupPolicy:
    confirmation: ""
  removeOSDsIfOutAndSafeToRemove: false
  storage:
    useAllNodes: true
    useAllDevices: false
    deviceFilter: "^sd[bcd]"    # 只使用 sdb, sdc, sdd
  disruptionManagement:
    managePodBudgets: true
    osdMaintenanceTimeout: 30
    pgHealthCheckTimeout: 0
  healthCheck:
    daemonHealth:
      mon:
        disabled: false
        interval: 45s
      osd:
        disabled: false
        interval: 60s
      status:
        disabled: false
        interval: 60s
    livenessProbe:
      mon:
        disabled: false
      mgr:
        disabled: false
      osd:
        disabled: false
```

```bash
kubectl apply -f ceph-cluster.yaml
kubectl -n rook-ceph get cephcluster
watch kubectl -n rook-ceph get pods
```

### 3.3 Rook StorageClass 建立

```yaml
# rook-storageclass-rbd.yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
    requireSafeReplicaSize: true
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions: []
```

---

## 4. K8s 串接外部 CEPH（主要方案）

### 4.1 收集 CEPH 叢集資訊

```bash
# 在 ceph1 上執行，收集以下資訊

# 1. 叢集 FSID
CEPH_FSID=$(ceph fsid)
echo "FSID: $CEPH_FSID"

# 2. MON 位址
ceph mon dump | grep "^[0-9]"
# 輸出範例：
# 0: [v2:192.168.100.31:3300/0,v1:192.168.100.31:6789/0] mon.ceph1

# 3. admin key（Base64 編碼用於 K8s Secret）
ADMIN_KEY=$(ceph auth get-key client.admin)
echo "Admin Key: $ADMIN_KEY"

# 4. CSI user key
CSI_RBD_KEY=$(ceph auth get-key client.kubernetes-csi)
echo "CSI RBD Key: $CSI_RBD_KEY"

CSI_CEPHFS_KEY=$(ceph auth get-key client.cephfs-csi)
echo "CSI CephFS Key: $CSI_CEPHFS_KEY"
```

### 4.2 安裝 ceph-csi Driver（在 K8s 上）

```bash
# 在 K8s master 執行

# 添加 Helm repo
helm repo add ceph-csi https://ceph.github.io/csi-charts
helm repo update

# 建立 namespace
kubectl create namespace ceph-csi

# 確認可用版本
helm search repo ceph-csi/ceph-csi-rbd --versions | head -5
helm search repo ceph-csi/ceph-csi-cephfs --versions | head -5
```

### 4.3 安裝 RBD CSI Driver

```bash
# 建立 RBD CSI values 設定檔
cat > /tmp/csi-rbd-values.yaml << 'EOF'
csiConfig:
  - clusterID: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"  # 替換為你的 FSID
    monitors:
      - "192.168.100.31:6789"
      - "192.168.100.32:6789"
      - "192.168.100.33:6789"

provisioner:
  replicaCount: 2
  tolerations: []
  affinity: {}

nodeplugin:
  tolerations: []

rbac:
  create: true

serviceAccounts:
  nodeplugin:
    create: true
    name: rbd-csi-nodeplugin
  provisioner:
    create: true
    name: rbd-csi-provisioner
EOF

# 安裝 RBD CSI
helm install ceph-csi-rbd ceph-csi/ceph-csi-rbd \
  --namespace ceph-csi \
  --version 3.11.0 \
  -f /tmp/csi-rbd-values.yaml

# 確認安裝狀態
kubectl -n ceph-csi get pods -l app=ceph-csi-rbd
```

### 4.4 安裝 CephFS CSI Driver

```bash
# 建立 CephFS CSI values 設定檔
cat > /tmp/csi-cephfs-values.yaml << 'EOF'
csiConfig:
  - clusterID: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"  # 替換為你的 FSID
    monitors:
      - "192.168.100.31:6789"
      - "192.168.100.32:6789"
      - "192.168.100.33:6789"
    cephFS:
      subvolumeGroup: "csi"

provisioner:
  replicaCount: 2

nodeplugin:
  tolerations: []

rbac:
  create: true
EOF

# 安裝 CephFS CSI
helm install ceph-csi-cephfs ceph-csi/ceph-csi-cephfs \
  --namespace ceph-csi \
  --version 3.11.0 \
  -f /tmp/csi-cephfs-values.yaml

# 確認安裝狀態
kubectl -n ceph-csi get pods -l app=ceph-csi-cephfs
```

### 4.5 建立 K8s Secret（儲存 CEPH 金鑰）

```bash
# 在 K8s master 執行
# 將 CEPH key 儲存到 K8s Secret（提供給 CSI 使用）

# RBD CSI Secret（Provisioner 用）
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: csi-rbd-secret
  namespace: default
stringData:
  # 替換為實際的 CEPH key
  userID: kubernetes-csi
  userKey: AQCxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==
EOF

# RBD CSI Secret（Node 用）
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: csi-rbd-node-secret
  namespace: default
stringData:
  userID: kubernetes-csi
  userKey: AQCxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==
EOF

# CephFS CSI Secret
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: csi-cephfs-secret
  namespace: default
stringData:
  adminID: admin
  adminKey: AQCxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==
  userID: cephfs-csi
  userKey: AQCxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==
EOF
```

### 4.6 建立 StorageClass（RBD）

```yaml
# storageclass-rbd.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # 設為預設 StorageClass
provisioner: rbd.csi.ceph.com
parameters:
  clusterID: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"   # 替換為你的 FSID
  pool: kubernetes-rbd
  imageFormat: "2"
  imageFeatures: "layering"
  csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
  csi.storage.k8s.io/provisioner-secret-namespace: default
  csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret
  csi.storage.k8s.io/controller-expand-secret-namespace: default
  csi.storage.k8s.io/node-stage-secret-name: csi-rbd-node-secret
  csi.storage.k8s.io/node-stage-secret-namespace: default
  csi.storage.k8s.io/fstype: ext4
reclaimPolicy: Delete       # PVC 刪除後自動刪除底層 RBD image
allowVolumeExpansion: true  # 允許動態擴容
volumeBindingMode: Immediate
mountOptions:
  - discard                 # 啟用 TRIM/discard，讓 CEPH 回收空間
```

```bash
kubectl apply -f storageclass-rbd.yaml
kubectl get storageclass
```

### 4.7 建立 StorageClass（CephFS）

```yaml
# storageclass-cephfs.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-cephfs
provisioner: cephfs.csi.ceph.com
parameters:
  clusterID: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"   # 替換為你的 FSID
  fsName: kubernetes-cephfs
  csi.storage.k8s.io/provisioner-secret-name: csi-cephfs-secret
  csi.storage.k8s.io/provisioner-secret-namespace: default
  csi.storage.k8s.io/controller-expand-secret-name: csi-cephfs-secret
  csi.storage.k8s.io/controller-expand-secret-namespace: default
  csi.storage.k8s.io/node-stage-secret-name: csi-cephfs-secret
  csi.storage.k8s.io/node-stage-secret-namespace: default
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
mountOptions:
  - debug                   # 除錯時啟用；生產環境請移除
```

```bash
kubectl apply -f storageclass-cephfs.yaml
kubectl get storageclass
# 輸出範例：
# NAME               PROVISIONER           RECLAIMPOLICY   VOLUMEBINDINGMODE   AGE
# ceph-rbd (default) rbd.csi.ceph.com      Delete          Immediate           2m
# ceph-cephfs        cephfs.csi.ceph.com   Delete          Immediate           1m
```

### 4.8 測試 PVC 建立與掛載

#### 測試 RBD PVC（ReadWriteOnce）

```yaml
# test-rbd-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-rbd-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: ceph-rbd
---
apiVersion: v1
kind: Pod
metadata:
  name: test-rbd-pod
  namespace: default
spec:
  containers:
    - name: test
      image: busybox:latest
      command: ["/bin/sh", "-c", "echo 'RBD test success' > /data/test.txt && sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: test-rbd-pvc
```

```bash
kubectl apply -f test-rbd-pvc.yaml

# 等待 PVC 綁定
kubectl get pvc test-rbd-pvc
# STATUS 應為 Bound

# 確認 Pod 狀態
kubectl get pod test-rbd-pod

# 確認資料寫入
kubectl exec test-rbd-pod -- cat /data/test.txt
# 輸出：RBD test success

# 清除測試資源
kubectl delete -f test-rbd-pvc.yaml
```

#### 測試 CephFS PVC（ReadWriteMany）

```yaml
# test-cephfs-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-cephfs-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteMany           # CephFS 支援多 Pod 同時寫入
  resources:
    requests:
      storage: 2Gi
  storageClassName: ceph-cephfs
---
# 部署兩個 Pod 同時掛載同一個 PVC（測試 RWX）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-cephfs-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cephfs-test
  template:
    metadata:
      labels:
        app: cephfs-test
    spec:
      containers:
        - name: test
          image: busybox:latest
          command: ["/bin/sh", "-c", "while true; do echo $(hostname) $(date) >> /shared/log.txt; sleep 5; done"]
          volumeMounts:
            - name: shared
              mountPath: /shared
      volumes:
        - name: shared
          persistentVolumeClaim:
            claimName: test-cephfs-pvc
```

```bash
kubectl apply -f test-cephfs-pvc.yaml

# 等待 PVC 和 Pod 就緒
kubectl get pvc test-cephfs-pvc
kubectl get pods -l app=cephfs-test

# 確認兩個 Pod 都在寫入同一個 PVC
kubectl exec deploy/test-cephfs-deploy -- cat /shared/log.txt

# 清除
kubectl delete -f test-cephfs-pvc.yaml
```

---

## 5. 效能調優建議

### 5.1 OSD 磁碟選擇

| 磁碟類型 | 適用場景 | 建議用途 |
|----------|----------|----------|
| **NVMe SSD** | 高 IOPS 需求（資料庫） | OSD data + WAL + DB 同碟 |
| **SSD** | 中等 IOPS | OSD data，或作為 HDD 的 WAL/DB 加速 |
| **HDD** | 大容量低成本 | OSD data；搭配 SSD 做 WAL/DB |

```bash
# 若使用 HDD + SSD 混合配置
# 在 SSD 上建立 WAL 和 DB 裝置
ceph orch daemon add osd ceph1:/dev/sdb --method raw \
  --db-devices /dev/nvme0n1 \
  --wal-devices /dev/nvme0n1
```

### 5.2 CRUSH Map 調整

```bash
# 確認預設 CRUSH rule
ceph osd crush rule ls
ceph osd crush rule dump

# 建立自訂 CRUSH rule（按 host 分散，確保 3 副本在不同節點）
ceph osd crush rule create-replicated replicated_rule default host

# 為 Pool 套用自訂規則
ceph osd pool set kubernetes-rbd crush_rule replicated_rule

# 調整 PG 數量（根據 OSD 數量調整）
# 建議：OSD 數 * 100 / 副本數，向上取整到 2 的次方
# 9 OSD, 3 replicas: 9 * 100 / 3 = 300 -> 取 256 或 512
ceph osd pool set kubernetes-rbd pg_num 256
ceph osd pool set kubernetes-rbd pgp_num 256

# 啟用 PG autoscaler（Reef 預設已啟用）
ceph osd pool set kubernetes-rbd pg_autoscale_mode on
```

### 5.3 K8s StorageClass reclaimPolicy 說明

| reclaimPolicy | 行為 | 適用場景 |
|---------------|------|----------|
| **Delete** | PVC 刪除後，底層 PV 及 CEPH 中的資料一併刪除 | 臨時資料、測試環境 |
| **Retain** | PVC 刪除後，PV 保留（狀態變為 Released），CEPH 資料保留 | 生產環境重要資料 |
| **Recycle** | 已棄用，不建議使用 | - |

```yaml
# 生產環境建議使用 Retain
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd-retain
provisioner: rbd.csi.ceph.com
parameters:
  # ... 同前
reclaimPolicy: Retain         # 生產環境建議
allowVolumeExpansion: true    # 允許擴容但不允許縮容
```

### 5.4 其他調優參數

```bash
# 調整 OSD 記憶體上限（預設 4GB）
ceph config set osd osd_memory_target 4294967296  # 4GB

# 調整 RBD cache 設定（提升讀取效能）
ceph config set client rbd_cache true
ceph config set client rbd_cache_size 134217728    # 128MB

# 啟用 messenger v2 加密（若有安全需求）
ceph config set global ms_bind_msgr2 true

# 調整 MON 設定
ceph config set mon auth_allow_insecure_global_id_reclaim false

# 確認所有設定
ceph config dump
```

---

## 附錄：常見問題排查

### PVC 長時間處於 Pending 狀態

```bash
# 查看 PVC 事件
kubectl describe pvc <pvc-name>

# 查看 CSI provisioner log
kubectl -n ceph-csi logs -l app=ceph-csi-rbd -c csi-provisioner --tail=50

# 確認 CEPH 叢集健康
ceph health detail

# 確認 Secret 中的 key 正確
kubectl get secret csi-rbd-secret -o jsonpath='{.data.userKey}' | base64 -d
```

### OSD 下線排查

```bash
# 查看 OSD 狀態
ceph osd tree
ceph health detail

# 查看特定 OSD 日誌
ceph log last 50 | grep osd

# 嘗試重啟 OSD daemon
ceph orch daemon restart osd.<id>
```

### 容量不足警告

```bash
# 查看容量使用率
ceph df
ceph osd df

# 設定容量警告閾值（預設 85%/95%）
ceph config set global mon_osd_full_ratio 0.90
ceph config set global mon_osd_backfillfull_ratio 0.85
ceph config set global mon_osd_nearfull_ratio 0.80
```
