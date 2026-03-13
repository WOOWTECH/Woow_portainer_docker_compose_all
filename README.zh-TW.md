# Portainer Docker Compose（相容 Podman）

使用 Podman 或 Docker 部署 [Portainer CE](https://www.portainer.io/)，支援 HTTP 存取，方便透過區域網路或 VPN 管理容器。

[English Documentation](README.md)

## 概述

本專案提供一個可直接用於生產環境的 `docker-compose.yml`，用於部署 Portainer CE：

- 開放 HTTP 連接埠 **9000**（透過區域網路/VPN 存取不會出現憑證警告）
- 開放 HTTPS 連接埠 **9443**（使用自簽憑證，選用）
- 以 **privileged 模式** 執行，讓 Portainer 可以完整管理容器（啟動、停止、重啟、刪除等）
- 停用 SELinux 標籤，防止掛載磁碟區時出現權限錯誤
- 支援 **Podman**（rootless 模式）及 **Docker**

## 系統需求

| 需求 | 最低版本 |
|---|---|
| Podman | 4.0+ |
| podman-compose | 1.0+ |
| **或** Docker | 20.10+ |
| docker-compose | 2.0+ |

## 快速開始

### 1. 複製儲存庫

```bash
git clone https://github.com/WOOWTECH/Woow_portainer_docker_compose_all.git
cd Woow_portainer_docker_compose_all
```

### 2. 調整 Podman Socket 路徑（如有需要）

預設的 `docker-compose.yml` 使用 UID 為 `1000` 的 rootless Podman socket：

```yaml
volumes:
  - /run/user/1000/podman/podman.sock:/var/run/docker.sock
```

**查詢你的 Socket 路徑：**

```bash
# 查詢你的使用者 ID
id -u

# 確認 socket 存在
ls -la /run/user/$(id -u)/podman/podman.sock
```

如果你的使用者 ID 不是 `1000`，請在 `docker-compose.yml` 中更新路徑：

```yaml
volumes:
  - /run/user/<你的UID>/podman/podman.sock:/var/run/docker.sock
```

**Docker 使用者**，請將磁碟區改為：

```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock
```

### 3. 部署

**使用 Podman：**

```bash
podman-compose up -d
```

**使用 Docker：**

```bash
docker compose up -d
```

### 4. 存取 Portainer

| 協定 | 網址 | 備註 |
|---|---|---|
| HTTP | `http://<你的IP>:9000` | 無憑證警告 |
| HTTPS | `https://<你的IP>:9443` | 自簽憑證，瀏覽器會顯示警告 |

將 `<你的IP>` 替換為你的伺服器區域網路 IP 或 VPN IP（例如 Tailscale）。

### 5. 首次設定

首次存取時，Portainer 會要求你：

1. 建立管理員帳號（帳號 + 密碼）
2. 選擇環境類型 — 選擇 **「Get Started」** 即可管理本機容器

## 檔案結構

```
.
├── docker-compose.yml    # 主要 Compose 設定檔
├── README.md             # 英文說明文件
├── README.zh-TW.md       # 繁體中文說明文件
└── docs/
    └── deployment.md     # 詳細部署指南
```

## docker-compose.yml 說明

```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest    # Portainer 社群版
    container_name: portainer
    restart: always                         # 故障或重開機後自動重啟
    privileged: true                        # 需要此權限才能完整管理容器
    security_opt:
      - label=disable                       # 停用 SELinux 標籤
    ports:
      - "9000:9000"                         # HTTP - 區域網路/VPN 友善
      - "9443:9443"                         # HTTPS - 自簽憑證
    volumes:
      - /run/user/1000/podman/podman.sock:/var/run/docker.sock   # Podman socket
      - portainer_data:/data                # 持久化資料

volumes:
  portainer_data:                           # Portainer 資料命名磁碟區
```

### 為什麼需要這些設定？

| 設定 | 原因 |
|---|---|
| `privileged: true` | 允許 Portainer 啟動、停止、重啟、刪除主機上的容器 |
| `security_opt: label=disable` | 防止 SELinux 阻擋被管理容器的磁碟區掛載 |
| 連接埠 `9000`（HTTP） | 避免透過區域網路/VPN 存取時出現 `NET::ERR_CERT_AUTHORITY_INVALID` 錯誤 |
| 連接埠 `9443`（HTTPS） | 選用的安全存取（使用自簽憑證） |
| `restart: always` | 主機重開機後容器自動啟動 |

## 常見問題

### 容器管理失敗，出現「permission denied」

Portainer 無法啟動/停止其他容器。

**原因：** 缺少 `privileged: true` 或 socket 路徑錯誤。

**解決方法：**
```bash
# 確認 socket 存在
ls -la /run/user/$(id -u)/podman/podman.sock

# 確認 docker-compose.yml 中已設定 privileged 模式
# 重新部署
podman-compose down && podman-compose up -d
```

### 被管理的容器啟動失敗，出現「cannot stat」或「No such file or directory」

**原因：** 容器有磁碟區綁定掛載指向主機上不存在的目錄。

**解決方法：** 建立缺少的目錄：
```bash
mkdir -p /path/to/missing/directory
```

然後在 Portainer 中重新啟動該容器。

### 存取連接埠 9443 時出現憑證警告

**原因：** Portainer 在 HTTPS 上使用自簽憑證。

**解決方法：** 改用 HTTP 連接埠 9000：
```
http://<你的IP>:9000
```

### 找不到 Podman Socket

**原因：** Podman socket 服務未執行。

**解決方法：**
```bash
# 啟用並啟動 Podman socket（rootless 模式）
systemctl --user enable podman.socket
systemctl --user start podman.socket

# 驗證
ls -la /run/user/$(id -u)/podman/podman.sock
```

## 透過 Portainer 管理容器

部署完成後，Portainer 提供網頁介面讓你：

- **啟動 / 停止 / 重啟 / 暫停 / 終止 / 移除** 容器
- **查看日誌** 及 **檢視** 容器詳細資訊
- 使用 Docker Compose 檔案 **部署堆疊（Stacks）**
- **管理磁碟區、網路和映像檔**
- **監控** 資源使用情況（CPU、記憶體、網路）

## 停止 / 移除 Portainer

```bash
# 停止
podman-compose down

# 停止並刪除資料磁碟區（警告：會刪除所有 Portainer 設定）
podman-compose down -v
```

## 授權條款

本專案為開放原始碼。Portainer CE 採用 [zlib 授權條款](https://github.com/portainer/portainer/blob/develop/LICENSE)。
