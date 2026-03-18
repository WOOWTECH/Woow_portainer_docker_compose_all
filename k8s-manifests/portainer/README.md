# Portainer K3s/Kubernetes 部署指南

[English](#english) | [中文](#中文)

---

## English

### Overview

Web-based container management UI for Docker, Kubernetes, and Swarm environments. Portainer Community Edition provides a visual dashboard for managing containers, images, networks, volumes, and Kubernetes resources without needing to memorize complex CLI commands. This deployment includes a ServiceAccount with `cluster-admin` privileges for full Kubernetes management capabilities.

> **GitHub Repo (Podman/Docker):** [Woow_portainer_docker_compose_all](https://github.com/WOOWTECH/Woow_portainer_docker_compose_all)

### Architecture

```
                     ┌─────────────────────────────────────────────┐
                     │              External Access                │
                     │  HTTPS: https://<node-ip>:30443             │
                     │  HTTP:  http://<node-ip>:30090              │
                     │  Edge:  <node-ip>:30800                     │
                     └────────────────┬────────────────────────────┘
                                      │
                                      ▼
                     ┌─────────────────────────────────────────────┐
                     │          NodePort Service                   │
                     │     portainer                               │
                     │     30443 → 9443 (HTTPS)                    │
                     │     30090 → 9000 (HTTP)                     │
                     │     30800 → 8000 (Edge Agent)               │
                     └────────────────┬────────────────────────────┘
                                      │
                                      ▼
                     ┌─────────────────────────────────────────────┐
                     │          ClusterIP Service                  │
                     │  portainer.portainer.svc:9443 (HTTPS)       │
                     │  portainer.portainer.svc:9000 (HTTP)        │
                     └────────────────┬────────────────────────────┘
                                      │
                                      ▼
              ┌────────────────────────────────────────────────────┐
              │           Deployment: portainer                    │
              │     ┌──────────────────────────────────┐           │
              │     │  portainer/portainer-ce:latest    │           │
              │     │  Ports: 9443, 9000, 8000         │           │
              │     │  Self-signed TLS certificate     │           │
              │     └────────┬─────────────────────────┘           │
              │              │                                     │
              │     ┌────────▼─────────────────────────┐           │
              │     │  PVC: portainer-data (10Gi)      │           │
              │     │  /data                           │           │
              │     │  (users, endpoints, settings,    │           │
              │     │   TLS certificates)              │           │
              │     └──────────────────────────────────┘           │
              └────────────────────────┬───────────────────────────┘
                                       │
                          ServiceAccount: portainer-sa
                          ClusterRoleBinding: portainer-crb
                                       │
                                       ▼
              ┌────────────────────────────────────────────────────┐
              │        Kubernetes API Server                       │
              │  (cluster-admin access to all namespaces)          │
              └────────────────────────────────────────────────────┘
```

### Features

- Visual dashboard for managing all Kubernetes resources
- Full cluster-admin access via RBAC ServiceAccount
- Self-signed HTTPS out of the box
- Edge Agent support for managing remote environments
- Container, image, network, and volume management
- Built-in web terminal for container exec
- Application template marketplace

### Quick Start

```bash
# 1. Deploy Portainer (includes ServiceAccount, RBAC, and PVC)
kubectl apply -k k8s-manifests/portainer/

# 2. Verify pods are running
kubectl -n portainer get pods

# 3. Access the web UI (create admin account within a few minutes)
echo "https://<node-ip>:30443"
```

### Configuration

#### No Environment Variables or Secrets Required

Portainer is configured through its web interface after initial deployment. No ConfigMap or Secret files are used.

#### RBAC Configuration

The deployment creates:
- **ServiceAccount:** `portainer-sa` in the `portainer` namespace
- **ClusterRoleBinding:** `portainer-crb` binding `portainer-sa` to the `cluster-admin` ClusterRole

This grants Portainer full access to manage all Kubernetes resources across all namespaces.

### Accessing the Service

| Endpoint | URL | Protocol |
|----------|-----|----------|
| Portainer Web UI (HTTPS) | `https://<node-ip>:30443` | HTTPS (NodePort) |
| Portainer Web UI (HTTP) | `http://<node-ip>:30090` | HTTP (NodePort) |
| Edge Agent | `<node-ip>:30800` | TCP (NodePort) |
| Internal HTTPS | `https://portainer.portainer.svc.cluster.local:9443` | HTTPS |
| Internal HTTP | `http://portainer.portainer.svc.cluster.local:9000` | HTTP |

On first access, you will be prompted to create an admin account and password. This must be done within a few minutes of deployment or Portainer will require a restart.

### Data Persistence

| PVC Name | Mount Path | Size | Purpose |
|----------|------------|------|---------|
| `portainer-data` | `/data` | 10Gi | User accounts, endpoint configurations, settings, TLS certificates |

The PVC uses the `local-path` storage class (k3s default).

### Backup & Restore

#### Backup

```bash
# Backup Portainer data (settings, users, endpoints)
kubectl -n portainer exec deploy/portainer -- tar czf /tmp/portainer-backup.tar.gz /data
kubectl -n portainer cp portainer/<pod-name>:/tmp/portainer-backup.tar.gz ./portainer-backup.tar.gz
```

#### Restore

```bash
# Restore Portainer data
kubectl -n portainer cp ./portainer-backup.tar.gz portainer/<pod-name>:/tmp/portainer-backup.tar.gz
kubectl -n portainer exec deploy/portainer -- tar xzf /tmp/portainer-backup.tar.gz -C /

# Restart Portainer
kubectl -n portainer rollout restart deploy/portainer
```

### Useful Commands

```bash
# Check pod status
kubectl -n portainer get pods

# View logs
kubectl -n portainer logs deploy/portainer -f

# Verify RBAC configuration
kubectl get serviceaccount portainer-sa -n portainer
kubectl get clusterrolebinding portainer-crb

# Restart the service
kubectl -n portainer rollout restart deploy/portainer

# Check all exposed ports
kubectl -n portainer get svc portainer

# Describe pod for detailed status
kubectl -n portainer describe pod -l app=portainer
```

### Troubleshooting

#### Timed out creating admin account

If you do not create the admin account within a few minutes of first deployment, Portainer will show a timeout error. Fix by restarting:

```bash
kubectl -n portainer rollout restart deploy/portainer
```

#### Cannot connect to the Kubernetes cluster

Verify the ServiceAccount and ClusterRoleBinding exist:

```bash
kubectl get serviceaccount portainer-sa -n portainer
kubectl get clusterrolebinding portainer-crb
```

#### HTTPS certificate warning

Portainer uses a self-signed TLS certificate by default. This is expected. You can:
- Accept the browser warning
- Configure a trusted certificate through the Portainer UI under Settings > SSL
- Use Nginx Proxy Manager to terminate TLS with a Let's Encrypt certificate

#### Pod not starting

```bash
kubectl -n portainer describe pod -l app=portainer
kubectl -n portainer logs deploy/portainer
```

#### Edge Agent connection issues

The Edge Agent port (30800) must be accessible from remote environments. Ensure firewall rules allow inbound connections on this port if using Edge Agents.

### File Structure

```
portainer/
├── kustomization.yaml          # Kustomize orchestration
├── namespace.yaml              # Namespace: portainer
├── serviceaccount.yaml         # ServiceAccount + ClusterRoleBinding (cluster-admin)
├── portainer-deployment.yaml   # Deployment with single replica
├── portainer-service.yaml      # NodePort (30443/30090/30800)
├── pvc.yaml                    # PVC: portainer-data (10Gi)
└── README.md                   # This file
```

---

## 中文

### 概述

基於網頁的容器管理 UI，適用於 Docker、Kubernetes 和 Swarm 環境。Portainer 社群版提供視覺化儀表板，可管理容器、映像檔、網路、卷和 Kubernetes 資源，無需記憶複雜的 CLI 指令。本部署包含具有 `cluster-admin` 權限的 ServiceAccount，以提供完整的 Kubernetes 管理能力。

> **GitHub 儲存庫 (Podman/Docker)：** [Woow_portainer_docker_compose_all](https://github.com/WOOWTECH/Woow_portainer_docker_compose_all)

### 架構

```
                     ┌─────────────────────────────────────────────┐
                     │              外部存取                       │
                     │  HTTPS: https://<node-ip>:30443             │
                     │  HTTP:  http://<node-ip>:30090              │
                     │  Edge:  <node-ip>:30800                     │
                     └────────────────┬────────────────────────────┘
                                      │
                                      ▼
                     ┌─────────────────────────────────────────────┐
                     │          NodePort 服務                      │
                     │     portainer                               │
                     │     30443 → 9443 (HTTPS)                    │
                     │     30090 → 9000 (HTTP)                     │
                     │     30800 → 8000 (Edge Agent)               │
                     └────────────────┬────────────────────────────┘
                                      │
                                      ▼
                     ┌─────────────────────────────────────────────┐
                     │          ClusterIP 服務                     │
                     │  portainer.portainer.svc:9443 (HTTPS)       │
                     │  portainer.portainer.svc:9000 (HTTP)        │
                     └────────────────┬────────────────────────────┘
                                      │
                                      ▼
              ┌────────────────────────────────────────────────────┐
              │           Deployment: portainer                    │
              │     ┌──────────────────────────────────┐           │
              │     │  portainer/portainer-ce:latest    │           │
              │     │  連接埠: 9443, 9000, 8000         │           │
              │     │  自簽 TLS 憑證                    │           │
              │     └────────┬─────────────────────────┘           │
              │              │                                     │
              │     ┌────────▼─────────────────────────┐           │
              │     │  PVC: portainer-data (10Gi)      │           │
              │     │  /data                           │           │
              │     │  (使用者、端點、設定、             │           │
              │     │   TLS 憑證)                       │           │
              │     └──────────────────────────────────┘           │
              └────────────────────────┬───────────────────────────┘
                                       │
                          ServiceAccount: portainer-sa
                          ClusterRoleBinding: portainer-crb
                                       │
                                       ▼
              ┌────────────────────────────────────────────────────┐
              │        Kubernetes API Server                       │
              │  （對所有命名空間的 cluster-admin 存取權限）         │
              └────────────────────────────────────────────────────┘
```

### 功能特色

- 視覺化儀表板，管理所有 Kubernetes 資源
- 透過 RBAC ServiceAccount 取得完整的 cluster-admin 存取權限
- 開箱即用的自簽 HTTPS
- Edge Agent 支援，管理遠端環境
- 容器、映像檔、網路和卷管理
- 內建網頁終端機，用於容器 exec
- 應用程式範本市場

### 快速開始

```bash
# 1. 部署 Portainer（包含 ServiceAccount、RBAC 和 PVC）
kubectl apply -k k8s-manifests/portainer/

# 2. 確認 Pod 正在運行
kubectl -n portainer get pods

# 3. 存取網頁 UI（需在幾分鐘內建立管理員帳號）
echo "https://<node-ip>:30443"
```

### 設定

#### 無需環境變數或 Secret

Portainer 在初始部署後透過其網頁介面進行設定。不使用 ConfigMap 或 Secret 檔案。

#### RBAC 設定

部署會建立：
- **ServiceAccount：** `portainer` 命名空間中的 `portainer-sa`
- **ClusterRoleBinding：** `portainer-crb` 將 `portainer-sa` 綁定到 `cluster-admin` ClusterRole

這授予 Portainer 完整權限，可管理所有命名空間中的所有 Kubernetes 資源。

### 存取服務

| 端點 | URL | 協定 |
|------|-----|------|
| Portainer 網頁 UI (HTTPS) | `https://<node-ip>:30443` | HTTPS (NodePort) |
| Portainer 網頁 UI (HTTP) | `http://<node-ip>:30090` | HTTP (NodePort) |
| Edge Agent | `<node-ip>:30800` | TCP (NodePort) |
| 內部 HTTPS | `https://portainer.portainer.svc.cluster.local:9443` | HTTPS |
| 內部 HTTP | `http://portainer.portainer.svc.cluster.local:9000` | HTTP |

首次存取時，系統會提示您建立管理員帳號和密碼。這必須在部署後幾分鐘內完成，否則 Portainer 將需要重新啟動。

### 資料持久化

| PVC 名稱 | 掛載路徑 | 大小 | 用途 |
|-----------|----------|------|------|
| `portainer-data` | `/data` | 10Gi | 使用者帳號、端點設定、設定值、TLS 憑證 |

PVC 使用 `local-path` 儲存類別（k3s 預設）。

### 備份與還原

#### 備份

```bash
# 備份 Portainer 資料（設定、使用者、端點）
kubectl -n portainer exec deploy/portainer -- tar czf /tmp/portainer-backup.tar.gz /data
kubectl -n portainer cp portainer/<pod-name>:/tmp/portainer-backup.tar.gz ./portainer-backup.tar.gz
```

#### 還原

```bash
# 還原 Portainer 資料
kubectl -n portainer cp ./portainer-backup.tar.gz portainer/<pod-name>:/tmp/portainer-backup.tar.gz
kubectl -n portainer exec deploy/portainer -- tar xzf /tmp/portainer-backup.tar.gz -C /

# 重新啟動 Portainer
kubectl -n portainer rollout restart deploy/portainer
```

### 實用指令

```bash
# 檢查 Pod 狀態
kubectl -n portainer get pods

# 檢視日誌
kubectl -n portainer logs deploy/portainer -f

# 驗證 RBAC 設定
kubectl get serviceaccount portainer-sa -n portainer
kubectl get clusterrolebinding portainer-crb

# 重新啟動服務
kubectl -n portainer rollout restart deploy/portainer

# 檢查所有暴露的連接埠
kubectl -n portainer get svc portainer

# 詳細描述 Pod 狀態
kubectl -n portainer describe pod -l app=portainer
```

### 疑難排解

#### 建立管理員帳號逾時

若您在首次部署後幾分鐘內未建立管理員帳號，Portainer 將顯示逾時錯誤。透過重新啟動修復：

```bash
kubectl -n portainer rollout restart deploy/portainer
```

#### 無法連接到 Kubernetes 叢集

確認 ServiceAccount 和 ClusterRoleBinding 存在：

```bash
kubectl get serviceaccount portainer-sa -n portainer
kubectl get clusterrolebinding portainer-crb
```

#### HTTPS 憑證警告

Portainer 預設使用自簽 TLS 憑證。這是正常的。您可以：
- 在瀏覽器中接受警告
- 透過 Portainer UI 的 Settings > SSL 設定受信任的憑證
- 使用 Nginx Proxy Manager 以 Let's Encrypt 憑證終止 TLS

#### Pod 無法啟動

```bash
kubectl -n portainer describe pod -l app=portainer
kubectl -n portainer logs deploy/portainer
```

#### Edge Agent 連線問題

Edge Agent 連接埠（30800）必須可從遠端環境存取。若使用 Edge Agent，請確保防火牆規則允許此連接埠的入站連線。

### 檔案結構

```
portainer/
├── kustomization.yaml          # Kustomize 編排檔
├── namespace.yaml              # 命名空間: portainer
├── serviceaccount.yaml         # ServiceAccount + ClusterRoleBinding (cluster-admin)
├── portainer-deployment.yaml   # Deployment（單副本）
├── portainer-service.yaml      # NodePort (30443/30090/30800)
├── pvc.yaml                    # PVC: portainer-data (10Gi)
└── README.md                   # 本檔案
```
