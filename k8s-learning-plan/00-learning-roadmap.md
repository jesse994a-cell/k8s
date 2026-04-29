# Kubernetes 企業級叢集學習路線圖

> **環境**：Proxmox VE 8.x + Ubuntu 24.04 + K8s 3M/3W HA + Ceph + GitLab + ArgoCD
> **目標**：建立支援高併發、具備完整資安防護的生產級 Kubernetes 叢集
> **撰寫日期**：2026-04-29

---

## 目錄

1. [學習路線圖總覽](#1-學習路線圖總覽)
2. [整體架構圖](#2-整體架構圖)
3. [硬體需求規格表](#3-硬體需求規格表)
4. [網路規劃](#4-網路規劃)
5. [學習時程規劃](#5-學習時程規劃)
6. [章節目錄與說明](#6-章節目錄與說明)
7. [前置知識需求](#7-前置知識需求)
8. [學習資源參考](#8-學習資源參考)

---

## 1. 學習路線圖總覽

本學習計畫共分為 **10 個章節**，從基礎環境建置到生產級資安防護，循序漸進地建立完整的企業級 Kubernetes 叢集。

### 章節概覽

| 章節 | 主題 | 難度 | 預估天數 |
|------|------|------|----------|
| 00 | 學習路線圖總覽（本文件） | 入門 | 1 天 |
| 01 | Proxmox VE 環境建置與 VM 建立 | 初級 | 3 天 |
| 02 | Ubuntu 24.04 基礎配置 | 初級 | 2 天 |
| 03 | Kubernetes HA 叢集安裝（kubeadm） | 中級 | 3 天 |
| 04 | Ceph 儲存叢集建置與 Rook 整合 | 中級 | 4 天 |
| 05 | 網路插件（Cilium/Calico）與 Network Policy | 中級 | 3 天 |
| 06 | GitLab 安裝與 CI/CD Pipeline 設計 | 中級 | 3 天 |
| 07 | ArgoCD GitOps 部署流程 | 中級 | 2 天 |
| 08 | RBAC 與身份認證授權 | 進階 | 2 天 |
| 09 | 資安強化（Falco、Trivy、Vault） | 進階 | 4 天 |
| 10 | 高併發優化、監控與維運 | 進階 | 4 天 |

**總學習時程估計：約 31 天（含練習與實作）**

---

## 2. 整體架構圖

### 2.1 叢集拓樸架構（ASCII Art）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Proxmox VE 超融合主機群                              │
│                                                                               │
│  ┌─── VLAN 10 (Management 192.168.10.0/24) ────────────────────────────┐    │
│  │   [Proxmox Node 1]   [Proxmox Node 2]   [Proxmox Node 3]             │    │
│  │   192.168.10.11      192.168.10.12      192.168.10.13                 │    │
│  └───────────────────────────────────────────────────────────────────────┘    │
│                                                                               │
│  ┌─── VLAN 20 (K8s Control Plane 192.168.20.0/24) ────────────────────┐     │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │     │
│  │   │  k8s-master1 │  │  k8s-master2 │  │  k8s-master3 │              │     │
│  │   │ 192.168.20.11│  │ 192.168.20.12│  │ 192.168.20.13│              │     │
│  │   │  4vCPU/8GB   │  │  4vCPU/8GB   │  │  4vCPU/8GB   │              │     │
│  │   └──────────────┘  └──────────────┘  └──────────────┘              │     │
│  │              HA VIP: 192.168.20.100 (keepalived + HAProxy)           │     │
│  └───────────────────────────────────────────────────────────────────────┘     │
│                                                                               │
│  ┌─── VLAN 30 (K8s Worker 192.168.30.0/24) ───────────────────────────┐     │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │     │
│  │   │  k8s-worker1 │  │  k8s-worker2 │  │  k8s-worker3 │              │     │
│  │   │ 192.168.30.11│  │ 192.168.30.12│  │ 192.168.30.13│              │     │
│  │   │  8vCPU/16GB  │  │  8vCPU/16GB  │  │  8vCPU/16GB  │              │     │
│  │   └──────────────┘  └──────────────┘  └──────────────┘              │     │
│  └───────────────────────────────────────────────────────────────────────┘     │
│                                                                               │
│  ┌─── VLAN 40 (Ceph Storage 192.168.40.0/24) ─────────────────────────┐     │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │     │
│  │   │  ceph-node1  │  │  ceph-node2  │  │  ceph-node3  │              │     │
│  │   │ 192.168.40.11│  │ 192.168.40.12│  │ 192.168.40.13│              │     │
│  │   │  4vCPU/8GB   │  │  4vCPU/8GB   │  │  4vCPU/8GB   │              │     │
│  │   │ 50GB+200GBx3 │  │ 50GB+200GBx3 │  │ 50GB+200GBx3 │              │     │
│  │   └──────────────┘  └──────────────┘  └──────────────┘              │     │
│  └───────────────────────────────────────────────────────────────────────┘     │
│                                                                               │
│  ┌─── VLAN 50 (CI/CD 192.168.50.0/24) ────────────────────────────────┐     │
│  │   ┌──────────────────────────────────────────┐                       │     │
│  │   │             gitlab-server                 │                       │     │
│  │   │  192.168.50.11 / 8vCPU / 16GB / 200GB    │                       │     │
│  │   └──────────────────────────────────────────┘                       │     │
│  └───────────────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 CI/CD 流程

```
開發者 → git push → GitLab → Pipeline 觸發
    → GitLab Runner → build image → Container Registry
    → Trivy 掃描 → 更新 GitOps Repo
    → ArgoCD Sync → Kubernetes 叢集部署
    → Falco 運行時監控 + Vault 機密管理
```

---

## 3. 硬體需求規格表

### 3.1 Proxmox Host 建議規格

| 項目 | 最低需求 | 建議配置 | 說明 |
|------|----------|----------|------|
| CPU | 16+ 核心，支援 VT-x/AMD-V | 雙路 CPU，32+ 核心 | 需支援硬體虛擬化 |
| RAM | 64 GB | 128 GB DDR4 ECC | 所有 VM 合計約需 112 GB |
| 系統磁碟 | 2 x SSD RAID-1 | 2 x NVMe 512GB RAID-1 (ZFS mirror) | |
| 資料磁碟 | SSD/NVMe | NVMe SSD（給 Ceph VM 磁碟） | |
| 網路 | 4 x 1GbE | 2 x 10GbE + 2 x 1GbE | 10GbE 給 Storage 和 VM 流量 |

### 3.2 VM 規格配置表

#### Master Nodes（x3）

| 參數 | 配置值 |
|------|--------|
| vCPU | 4 |
| RAM | 8 GB |
| 系統磁碟 | 80 GB（SSD/NVMe） |
| OS | Ubuntu 24.04 LTS |
| IP | 192.168.20.11 / .12 / .13 |

#### Worker Nodes（x3）

| 參數 | 配置值 |
|------|--------|
| vCPU | 8 |
| RAM | 16 GB |
| 系統磁碟 | 100 GB（SSD/NVMe） |
| OS | Ubuntu 24.04 LTS |
| IP | 192.168.30.11 / .12 / .13 |

#### Ceph Storage Nodes（x3）

| 參數 | 配置值 |
|------|--------|
| vCPU | 4 |
| RAM | 8 GB |
| 系統磁碟 | 50 GB |
| 資料磁碟 | 200 GB x 3（空白，供 Ceph OSD 使用） |
| OS | Ubuntu 24.04 LTS |
| IP（Public） | 192.168.40.11 / .12 / .13 |
| IP（Cluster） | 192.168.41.11 / .12 / .13 |

#### GitLab Server（x1）

| 參數 | 配置值 |
|------|--------|
| vCPU | 8 |
| RAM | 16 GB |
| 系統磁碟 | 200 GB |
| OS | Ubuntu 24.04 LTS |
| IP | 192.168.50.11 |

### 3.3 資源彙整

| 角色 | 數量 | vCPU | RAM | 磁碟 |
|------|------|------|-----|------|
| Master | 3 | 12 | 24 GB | 240 GB |
| Worker | 3 | 24 | 48 GB | 300 GB |
| Ceph | 3 | 12 | 24 GB | 150 GB + 1800 GB |
| GitLab | 1 | 8 | 16 GB | 200 GB |
| **合計** | **10** | **56** | **112 GB** | **2690 GB** |

---

## 4. 網路規劃

### 4.1 VLAN 規劃表

| VLAN ID | 用途 | 網段 | 閘道 |
|---------|------|------|------|
| VLAN 10 | Proxmox 管理網 | 192.168.10.0/24 | 192.168.10.1 |
| VLAN 20 | K8s Control Plane | 192.168.20.0/24 | 192.168.20.1 |
| VLAN 30 | K8s Worker | 192.168.30.0/24 | 192.168.30.1 |
| VLAN 40 | Ceph Public | 192.168.40.0/24 | 192.168.40.1 |
| VLAN 41 | Ceph Cluster | 192.168.41.0/24 | — |
| VLAN 50 | CI/CD | 192.168.50.0/24 | 192.168.50.1 |
| — | Pod Network（虛擬） | 10.244.0.0/16 | — |
| — | Service Network（虛擬） | 10.96.0.0/12 | — |

### 4.2 IP 位址配置表

| Hostname | 角色 | IP |
|----------|------|-----|
| k8s-master1 | K8s Master | 192.168.20.11 |
| k8s-master2 | K8s Master | 192.168.20.12 |
| k8s-master3 | K8s Master | 192.168.20.13 |
| k8s-api-vip | API Server VIP | 192.168.20.100 |
| k8s-worker1 | K8s Worker | 192.168.30.11 |
| k8s-worker2 | K8s Worker | 192.168.30.12 |
| k8s-worker3 | K8s Worker | 192.168.30.13 |
| ceph-node1 | Ceph | 192.168.40.11 / 192.168.41.11 |
| ceph-node2 | Ceph | 192.168.40.12 / 192.168.41.12 |
| ceph-node3 | Ceph | 192.168.40.13 / 192.168.41.13 |
| gitlab-server | GitLab | 192.168.50.11 |

---

## 5. 學習時程規劃

```
Week 1（Day 1-7）：基礎環境建置
  Day 1: 閱讀路線圖、了解架構、準備 Proxmox 環境
  Day 2: Proxmox 安裝、網路設定、VLAN 配置
  Day 3: 使用 cloud-init 批次建立所有 VM
  Day 4: Ubuntu 24.04 基礎配置（master nodes）
  Day 5: Ubuntu 24.04 基礎配置（worker + ceph nodes）
  Day 6: 安裝 containerd、kubeadm、kubelet、kubectl
  Day 7: 複習、驗證環境

Week 2（Day 8-14）：K8s 叢集建立
  Day 8:  初始化第一個 master，建立 HA 架構（keepalived）
  Day 9:  加入 master2、master3 到叢集
  Day 10: 加入所有 worker nodes
  Day 11: 安裝 Cilium CNI，驗證網路
  Day 12: Ceph 叢集建置（OSD、MON、MGR）
  Day 13: Rook-Ceph Operator 安裝與 StorageClass 建立
  Day 14: 驗證持久化儲存、整體叢集測試

Week 3（Day 15-21）：CI/CD 平台建置
  Day 15: GitLab 安裝（Omnibus 或 Helm chart）
  Day 16: GitLab Runner 配置
  Day 17: 建立第一個 CI Pipeline（build + test + scan）
  Day 18: ArgoCD 安裝與基礎設定
  Day 19: GitOps 工作流程設計與實作
  Day 20: 端對端 CI/CD 測試
  Day 21: 優化 Pipeline，加入 Trivy 掃描

Week 4-5（Day 22-31）：資安強化與效能優化
  Day 22: RBAC 設計與實作
  Day 23: Network Policy 設計與實作（Cilium Policy）
  Day 24: HashiCorp Vault 安裝與 K8s 整合
  Day 25: Falco 安裝與規則配置
  Day 26: Trivy Operator 安裝（叢集掃描模式）
  Day 27: Prometheus + Grafana 監控系統
  Day 28: HPA、VPA、PodDisruptionBudget 高可用設定
  Day 29: 壓力測試與效能調校
  Day 30: 災難復原演練（etcd 備份還原）
  Day 31: 整體複習、文件整理
```

---

## 6. 章節目錄與說明

| 檔案 | 主題 | 重點內容 |
|------|------|----------|
| 00-learning-roadmap.md | 本文件 | 架構總覽、規格、網路、時程 |
| 01-proxmox-vm-setup.md | Proxmox 環境建置 | PVE 安裝、VLAN 設定、cloud-init VM 建立、批次腳本 |
| 02-ubuntu24-base-config.md | Ubuntu 系統配置 | 系統初始化、kernel 參數、containerd、kubeadm |
| 03-k8s-ha-cluster-install.md | K8s HA 叢集安裝 | keepalived、kubeadm init、HA join |
| 04-ceph-rook-storage.md | Ceph 儲存建置 | MON/OSD/MGR、Rook Operator、StorageClass |
| 05-network-cni-policy.md | 網路插件與 Policy | Cilium 安裝、Hubble、Network Policy |
| 06-gitlab-cicd.md | GitLab CI/CD | Omnibus 安裝、Runner、Pipeline 設計 |
| 07-argocd-gitops.md | ArgoCD GitOps | 安裝、Application、Sync Policy |
| 08-rbac-auth.md | RBAC 與認證 | Role/ClusterRole、OIDC、kubeconfig |
| 09-security-hardening.md | 資安強化 | Falco、Trivy、Vault、Pod Security |
| 10-performance-monitoring.md | 效能監控維運 | Prometheus、HPA/VPA、etcd 備份 |

---

## 7. 前置知識需求

### 7.1 必備知識

| 領域 | 具體項目 | 熟悉程度 |
|------|----------|----------|
| Linux | 檔案系統、程序管理、網路指令 | 熟練 |
| 網路 | TCP/IP、VLAN、路由、防火牆 | 了解 |
| Docker | 容器基礎、Dockerfile、volumes | 熟練 |
| Git | 基本操作、branch、merge | 了解 |
| YAML | 語法格式、縮排規則 | 熟練 |

### 7.2 建議先修路徑

```
Linux 基礎
├── Bash scripting（迴圈、條件、函數）
├── systemd 服務管理
├── 網路工具（ip, ss, netstat, tcpdump）
└── 磁碟管理（lsblk, fdisk, lvm）

容器技術
├── Docker 基礎操作
├── docker-compose
├── Container networking
└── Container storage（volumes, bind mounts）

網路概念
├── OSI 7 層模型
├── VLAN 與 trunking
└── iptables/nftables 基礎
```

### 7.3 認證路徑建議

```
初級: KCNA（Kubernetes and Cloud Native Associate）
中級: CKAD（Application Developer）/ CKA（Administrator）
進階: CKS（Security Specialist，需先取得 CKA）
```

---

## 8. 學習資源參考

### 8.1 官方文件

- Kubernetes: https://kubernetes.io/docs/
- Proxmox VE: https://pve.proxmox.com/wiki/
- Ceph: https://docs.ceph.com/
- Cilium: https://docs.cilium.io/
- ArgoCD: https://argo-cd.readthedocs.io/

### 8.2 常用工具清單

| 工具 | 用途 |
|------|------|
| kubectl | K8s 命令列工具 |
| kubeadm | 叢集初始化工具 |
| helm | K8s 套件管理 |
| k9s | K8s TUI 管理工具 |
| kubectx/kubens | 快速切換 context/namespace |
| stern | 多 Pod 日誌追蹤 |
| velero | 叢集備份 |
| cilium CLI | Cilium 管理工具 |
| argocd CLI | ArgoCD 命令列工具 |
| trivy | 映像安全掃描 |

### 8.3 本計畫檔案結構

```
k8s-learning-plan/
├── 00-learning-roadmap.md          ← 本文件
├── 01-proxmox-vm-setup.md
├── 02-ubuntu24-base-config.md
├── 03-k8s-ha-cluster-install.md
├── 04-ceph-rook-storage.md
├── 05-network-cni-policy.md
├── 06-gitlab-cicd.md
├── 07-argocd-gitops.md
├── 08-rbac-auth.md
├── 09-security-hardening.md
├── 10-performance-monitoring.md
└── configs/
    ├── proxmox/
    ├── k8s/
    ├── ceph/
    └── security/
```

---

> **提示**：建議按照章節順序學習，每章節完成後進行實際操作驗證後再進入下一章節。
> **版本資訊**：Kubernetes v1.30+、Proxmox VE 8.x、Ubuntu 24.04 LTS、Ceph Reef（18.x）
