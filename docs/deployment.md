# Portainer Deployment Guide / Portainer 部署指南

A step-by-step skill reference for deploying Portainer CE on Podman or Docker with HTTP access for LAN/VPN.

---

## Skill Metadata

- **Name:** portainer-deployment
- **Trigger:** Deploy Portainer, set up container management UI, Portainer docker-compose
- **Type:** Rigid (follow steps exactly)

---

## Prerequisites Checklist

- [ ] Linux server (Ubuntu 22.04+, Fedora 38+, or similar)
- [ ] Podman 4.0+ with podman-compose **OR** Docker 20.10+ with docker-compose
- [ ] Podman socket enabled (for Podman users)
- [ ] Network access to ports 9000 and/or 9443

---

## Step-by-Step Deployment

### Step 1: Verify Podman Socket (Podman Only)

```bash
# Check if Podman socket is active
systemctl --user status podman.socket

# If not active, enable and start it
systemctl --user enable podman.socket
systemctl --user start podman.socket

# Verify socket file exists
ls -la /run/user/$(id -u)/podman/podman.sock
```

Expected output: `srw-------` socket file exists.

### Step 2: Clone the Repository

```bash
git clone https://github.com/WOOWTECH/Woow_portainer_docker_compose_all.git
cd Woow_portainer_docker_compose_all
```

### Step 3: Verify docker-compose.yml

Check that the socket path matches your system:

```bash
# Your user ID
id -u

# If output is NOT 1000, edit docker-compose.yml:
# Change: /run/user/1000/podman/podman.sock
# To:     /run/user/<YOUR_UID>/podman/podman.sock
```

For Docker users, change the socket path:
```yaml
# Podman (default):
- /run/user/1000/podman/podman.sock:/var/run/docker.sock

# Docker:
- /var/run/docker.sock:/var/run/docker.sock
```

### Step 4: Deploy

```bash
# Podman
podman-compose up -d

# Docker
docker compose up -d
```

### Step 5: Verify Deployment

```bash
# Podman
podman ps --filter name=portainer --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Docker
docker ps --filter name=portainer --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Expected: Container is `Up` with ports `0.0.0.0:9000->9000/tcp` and `0.0.0.0:9443->9443/tcp`.

### Step 6: Access Portainer

Open your browser:

```
http://<SERVER_IP>:9000
```

- For LAN access: use your server's local IP (e.g., `192.168.1.100`)
- For VPN access: use your VPN IP (e.g., Tailscale `100.x.x.x`)

### Step 7: Initial Configuration

1. Create admin account with a strong password (minimum 12 characters)
2. Click **"Get Started"** to connect to the local Docker/Podman environment
3. You should see your local environment listed with container stats

---

## Managing Other Containers

Once Portainer is running, you can manage all containers on the host through the web UI.

### Common Issue: "Cannot stat" Errors

When starting containers that have bind mounts, you may see:

```
crun: cannot stat `/path/to/directory`: No such file or directory
```

**Resolution:** Create the missing directory on the host:

```bash
mkdir -p /path/to/missing/directory
```

Then retry starting the container in Portainer.

### Common Issue: Missing Marker Files (Immich)

Immich requires `.immich` marker files in upload subdirectories:

```bash
UPLOAD_DIR="/path/to/immich/data/library"
for dir in thumbs upload backups profile encoded-video library; do
  mkdir -p "$UPLOAD_DIR/$dir"
  touch "$UPLOAD_DIR/$dir/.immich"
done
```

---

## Operations Reference

### Stop Portainer

```bash
podman-compose down       # Podman
docker compose down       # Docker
```

### Restart Portainer

```bash
podman-compose restart    # Podman
docker compose restart    # Docker
```

### View Logs

```bash
podman-compose logs -f    # Podman
docker compose logs -f    # Docker
```

### Update Portainer

```bash
podman-compose pull       # Pull latest image
podman-compose down       # Stop current
podman-compose up -d      # Start with new image
```

### Remove Portainer and All Data

```bash
podman-compose down -v    # WARNING: Deletes all Portainer settings
```

---

## Security Notes

- **HTTP (port 9000)** is unencrypted. Only use on trusted networks (LAN/VPN).
- **HTTPS (port 9443)** uses a self-signed certificate. Browsers will show a warning.
- `privileged: true` gives the container full host access. This is required for Portainer to manage other containers.
- Set a strong admin password on first login.
- Consider firewall rules to restrict access to trusted IPs only.

---

## Network Diagram

```
┌──────────────┐     HTTP :9000      ┌─────────────────┐
│   Browser    │ ──────────────────> │   Portainer CE   │
│ (LAN / VPN)  │     HTTPS :9443     │   Container      │
└──────────────┘ ──────────────────> │                  │
                                     │  podman.sock ──────> Host Podman/Docker
                                     │  portainer_data ──> Persistent Config
                                     └─────────────────┘
```
