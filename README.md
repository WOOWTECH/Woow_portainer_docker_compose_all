# Portainer Docker Compose (Podman Compatible)

Deploy [Portainer CE](https://www.portainer.io/) with HTTP access for LAN/VPN management using Podman or Docker.

[繁體中文說明](README.zh-TW.md)

## Overview

This project provides a production-ready `docker-compose.yml` for Portainer CE that:

- Exposes HTTP on port **9000** (no certificate warnings over LAN/VPN)
- Exposes HTTPS on port **9443** (self-signed cert, optional)
- Runs in **privileged mode** so Portainer can fully manage containers (start, stop, restart, delete, etc.)
- Disables SELinux labeling to prevent volume mount permission errors
- Works with **Podman** (rootless) and **Docker**

## Prerequisites

| Requirement | Minimum Version |
|---|---|
| Podman | 4.0+ |
| podman-compose | 1.0+ |
| **OR** Docker | 20.10+ |
| docker-compose | 2.0+ |

## Quick Start

### 1. Clone the repository

```bash
git clone https://github.com/WOOWTECH/Woow_portainer_docker_compose_all.git
cd Woow_portainer_docker_compose_all
```

### 2. Adjust the Podman socket path (if needed)

The default `docker-compose.yml` uses the rootless Podman socket for user ID `1000`:

```yaml
volumes:
  - /run/user/1000/podman/podman.sock:/var/run/docker.sock
```

**Find your socket path:**

```bash
# Check your user ID
id -u

# Verify the socket exists
ls -la /run/user/$(id -u)/podman/podman.sock
```

If your user ID is not `1000`, update the path in `docker-compose.yml`:

```yaml
volumes:
  - /run/user/<YOUR_UID>/podman/podman.sock:/var/run/docker.sock
```

**For Docker users**, change the volume to:

```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock
```

### 3. Deploy

**With Podman:**

```bash
podman-compose up -d
```

**With Docker:**

```bash
docker compose up -d
```

### 4. Access Portainer

| Protocol | URL | Notes |
|---|---|---|
| HTTP | `http://<YOUR_IP>:9000` | No certificate warnings |
| HTTPS | `https://<YOUR_IP>:9443` | Self-signed cert, browser warning |

Replace `<YOUR_IP>` with your server's LAN IP or VPN IP (e.g., Tailscale).

### 5. Initial Setup

On first access, Portainer will ask you to:

1. Create an admin account (username + password)
2. Select the environment type - choose **"Get Started"** for local management

## File Structure

```
.
├── docker-compose.yml    # Main compose file
├── README.md             # English documentation
├── README.zh-TW.md       # 繁體中文說明
└── docs/
    └── deployment.md     # Detailed deployment guide
```

## docker-compose.yml Explained

```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest    # Portainer Community Edition
    container_name: portainer
    restart: always                         # Auto-restart on failure/reboot
    privileged: true                        # Required for full container management
    security_opt:
      - label=disable                       # Disable SELinux labeling
    ports:
      - "9000:9000"                         # HTTP - LAN/VPN friendly
      - "9443:9443"                         # HTTPS - self-signed cert
    volumes:
      - /run/user/1000/podman/podman.sock:/var/run/docker.sock   # Podman socket
      - portainer_data:/data                # Persistent data

volumes:
  portainer_data:                           # Named volume for Portainer data
```

### Why These Settings?

| Setting | Reason |
|---|---|
| `privileged: true` | Allows Portainer to start/stop/restart/delete containers on the host |
| `security_opt: label=disable` | Prevents SELinux from blocking volume mounts in managed containers |
| Port `9000` (HTTP) | Avoids `NET::ERR_CERT_AUTHORITY_INVALID` errors when accessing over LAN/VPN |
| Port `9443` (HTTPS) | Optional secure access with self-signed certificate |
| `restart: always` | Container auto-starts after host reboot |

## Common Issues

### Container management fails with "permission denied"

Portainer cannot start/stop other containers.

**Cause:** Missing `privileged: true` or wrong socket path.

**Fix:**
```bash
# Verify socket exists
ls -la /run/user/$(id -u)/podman/podman.sock

# Ensure privileged mode is set in docker-compose.yml
# Redeploy
podman-compose down && podman-compose up -d
```

### Managed containers fail with "cannot stat" / "No such file or directory"

**Cause:** Containers have volume bind mounts pointing to directories that don't exist on the host.

**Fix:** Create the missing directories:
```bash
mkdir -p /path/to/missing/directory
```

Then retry starting the container in Portainer.

### Certificate warning when accessing port 9443

**Cause:** Portainer uses a self-signed certificate on HTTPS.

**Fix:** Use HTTP on port 9000 instead:
```
http://<YOUR_IP>:9000
```

### Podman socket not found

**Cause:** Podman socket service is not running.

**Fix:**
```bash
# Enable and start the Podman socket (rootless)
systemctl --user enable podman.socket
systemctl --user start podman.socket

# Verify
ls -la /run/user/$(id -u)/podman/podman.sock
```

## Managing Containers via Portainer

Once deployed, Portainer provides a web UI to:

- **Start / Stop / Restart / Pause / Kill / Remove** containers
- **View logs** and **inspect** container details
- **Deploy stacks** using Docker Compose files
- **Manage volumes, networks, and images**
- **Monitor** resource usage (CPU, memory, network)

## Stop / Remove Portainer

```bash
# Stop
podman-compose down

# Stop and remove data volume (WARNING: deletes all Portainer settings)
podman-compose down -v
```

## License

This project is open source. Portainer CE is licensed under the [zlib License](https://github.com/portainer/portainer/blob/develop/LICENSE).

---

## K3s/Kubernetes Deployment

This project also supports deployment on **K3s/Kubernetes** clusters. The K3s manifests are maintained on a separate branch.

### Quick Start (K3s)

```bash
# Clone the k3s branch
git clone -b k3s https://github.com/WOOWTECH/Woow_portainer_docker_compose_all.git Woow_portainer_docker_compose_all-k3s
cd Woow_portainer_docker_compose_all-k3s

# Edit secrets before deploying
nano secret.yaml

# Deploy to your k3s cluster
kubectl apply -k .

# Verify pods are running
kubectl -n portainer get pods
```

### Deployment Methods Comparison

| Feature | Podman/Docker Compose | K3s/Kubernetes |
|---------|----------------------|----------------|
| Branch | `main` | `k3s` |
| Orchestrator | Podman / Docker | K3s / Kubernetes |
| Config format | `.env` + `docker-compose.yml` | ConfigMap + Secret + YAML manifests |
| Scaling | Manual | `kubectl scale` |
| Health checks | Docker healthcheck | liveness/readiness/startup probes |
| Service discovery | Docker DNS | Kubernetes DNS (`svc.cluster.local`) |
| Storage | Docker volumes | PersistentVolumeClaims |
| Rolling updates | `docker compose pull && up -d` | `kubectl rollout restart` |

> For full K3s deployment documentation, switch to the [`k3s` branch](https://github.com/WOOWTECH/Woow_portainer_docker_compose_all/tree/k3s).
