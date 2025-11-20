# Deployment Guide

This guide covers the deployment and management of the Docker Compose homelab stack.

## Prerequisites

- Docker Engine 24.0 or later
- Docker Compose v2.20 or later
- Linux system with systemd
- Root or sudo access

## Initial Setup

### 1. Install Docker

```bash
# Install Docker Engine
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to docker group
sudo usermod -aG docker $USER

# Enable Docker service
sudo systemctl enable docker
sudo systemctl start docker
```

### 2. Clone Repository

```bash
# Create containers directory
sudo mkdir -p /srv/containers

# Clone repository
sudo git clone <repository-url> /srv/containers
cd /srv/containers

# Set proper ownership
sudo chown -R $USER:$USER /srv/containers
```

### 3. Create Data Directories

```bash
# Create appdata directory
sudo mkdir -p /var/lib/containers/appdata

# Set ownership (use your PUID/PGID)
sudo chown -R 1000:1000 /var/lib/containers/appdata
```

### 4. Configure Environment Variables

For each stack you want to deploy:

```bash
cd /srv/containers/[stack-name]
cp .env.example .env
nano .env  # Edit with your values
```

Common environment variables:
- `PUID`: Your user ID (run `id -u` to find)
- `PGID`: Your group ID (run `id -g` to find)
- `TZ`: Your timezone (e.g., `America/New_York`, `Europe/London`)

## Systemd Integration

To manage stacks with systemd, create service files for automatic startup and management.

### Creating a Systemd Service

Example for media stack:

```bash
sudo nano /etc/systemd/system/docker-compose-media.service
```

Add the following content:

```ini
[Unit]
Description=Docker Compose Media Stack
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/srv/containers/media
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```

Repeat for web and cloud stacks, changing the description and working directory.

### Enable and Start Services

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable services to start on boot
sudo systemctl enable docker-compose-media.service
sudo systemctl enable docker-compose-web.service
sudo systemctl enable docker-compose-nextcloud.service
sudo systemctl enable docker-compose-immich.service
# Note: immich-ml runs on fileserver, create service there

# Start services
sudo systemctl start docker-compose-media.service
sudo systemctl start docker-compose-web.service
sudo systemctl start docker-compose-nextcloud.service
sudo systemctl start docker-compose-immich.service

# Check status
sudo systemctl status docker-compose-media.service
```

## Stack Management

### Starting a Stack

```bash
cd /srv/containers/[stack-name]
docker compose up -d
```

### Stopping a Stack

```bash
cd /srv/containers/[stack-name]
docker compose down
```

### Viewing Logs

```bash
cd /srv/containers/[stack-name]
docker compose logs -f [service-name]
```

### Updating Containers

```bash
cd /srv/containers/[stack-name]
docker compose pull
docker compose up -d
```

### Restarting a Service

```bash
cd /srv/containers/[stack-name]
docker compose restart [service-name]
```

## Backup and Restore

### Backing Up Configuration

```bash
# Backup compose files and environment
cd /srv/containers
tar -czf ~/homelab-config-backup-$(date +%F).tar.gz \
  --exclude='.git' \
  media/ web/ cloud/ README.md DEPLOYMENT.md

# Backup appdata
sudo tar -czf ~/homelab-appdata-backup-$(date +%F).tar.gz \
  -C /var/lib/containers appdata/
```

### Restoring from Backup

```bash
# Restore configuration
sudo tar -xzf ~/homelab-config-backup-*.tar.gz -C /srv/containers

# Restore appdata
sudo tar -xzf ~/homelab-appdata-backup-*.tar.gz -C /var/lib/containers
sudo chown -R 1000:1000 /var/lib/containers/appdata
```

## Troubleshooting

### Check Container Status

```bash
cd /srv/containers/[stack-name]
docker compose ps
```

### View Container Logs

```bash
docker compose logs -f [service-name]
```

### Recreate Containers

```bash
docker compose down
docker compose up -d --force-recreate
```

### Check Permissions

```bash
ls -la /var/lib/containers/appdata/[service-name]
```

Ensure directories are owned by the user specified in PUID/PGID.

### Network Issues

```bash
# Check Docker networks
docker network ls

# Inspect network
docker network inspect [stack-name]_default
```

## Security Considerations

1. **Environment Files**: Never commit `.env` files to git - they contain sensitive data
2. **File Permissions**: Ensure proper ownership of appdata directories
3. **Network Exposure**: Use reverse proxy for external access
4. **Updates**: Regularly update containers with `docker compose pull`
5. **Firewall**: Configure firewall rules to restrict access

## Updating the Stack

To update stack configurations:

```bash
cd /srv/containers
git pull
cd [stack-name]
docker compose up -d
```

## Monitoring

Monitor your containers with:

```bash
# Resource usage
docker stats

# Container health
docker compose ps

# System logs
journalctl -u docker-compose-media.service -f
```

## Stack-Specific Deployment Notes

### Nextcloud All-in-One

Nextcloud uses a special AIO (All-in-One) container that manages additional containers automatically.

**Initial Setup:**
1. Deploy the stack: `cd /srv/containers/nextcloud && docker compose up -d`
2. Get the initial admin password: `docker logs nextcloud-aio-mastercontainer | grep password`
3. Access AIO interface: `http://YOUR_SERVER_IP:8080`
4. Follow the setup wizard to configure domain, SSL, and components
5. The mastercontainer will automatically create and manage all required containers

**Important Notes:**
- AIO creates containers outside of Docker Compose (managed via Docker API)
- These containers are prefixed with `nextcloud-aio-`
- Updates are handled through the AIO interface at port 8080
- See `nextcloud/README.md` for detailed configuration

**Storage:**
- Default: Uses Docker volumes managed by AIO
- For custom storage: Set `NEXTCLOUD_DATADIR` in `.env` before first start
- Example: `NEXTCLOUD_DATADIR=/mnt/nas-storage/nextcloud`

### Immich Split Deployment

Immich uses a **distributed deployment** across two hosts for optimal performance.

**Deployment Order:**

1. **First: Deploy ML Stack on Fileserver**

   The ML stack is deployed via Docker's web UI (Portainer, Yacht, etc.).

   See `immich-ml/DOCKER-UI-DEPLOYMENT.md` for complete instructions.

   Quick summary:
   - Deploy `ghcr.io/immich-app/immich-machine-learning:release` image
   - Enable GPU access (`--gpus all`)
   - Expose port 3003
   - Mount volume for model cache
   - Verify with: `nvidia-smi` and `docker logs immich_machine_learning`

2. **Second: Deploy Main Stack on Mini PC**
   ```bash
   # On mini PC
   cd /srv/containers/immich
   cp .env.example .env
   nano .env  # Set IMMICH_MACHINE_LEARNING_URL=http://FILESERVER_IP:3003

   # Create storage directories
   sudo mkdir -p /var/lib/containers/appdata/immich/{upload,postgres}
   sudo chown -R 1000:1000 /var/lib/containers/appdata/immich

   docker compose up -d
   ```

3. **Verify Connection**
   ```bash
   # From mini PC
   curl http://FILESERVER_IP:3003/ping

   # Check Immich logs
   docker logs immich_server
   ```

**Prerequisites for ML Stack (Fileserver):**
- NVIDIA drivers installed: `nvidia-smi` should work
- NVIDIA Container Toolkit installed
- Firewall allows port 3003 from mini PC IP

**Install NVIDIA Container Toolkit on Fileserver:**
```bash
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | \
  sudo tee /etc/apt/sources.list.d/nvidia-docker.list

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
sudo systemctl restart docker

# Verify
docker run --rm --gpus all nvidia/cuda:11.8.0-base-ubuntu22.04 nvidia-smi
```

**Firewall Configuration (Fileserver):**
```bash
# Allow mini PC to access ML service
sudo ufw allow from MINI_PC_IP to any port 3003
```

**Network Requirements:**
- Low latency between mini PC and fileserver (same LAN recommended)
- Sufficient bandwidth for image processing (1Gbps recommended)
- Static IPs or DNS names for stable connectivity

**Troubleshooting:**
- If ML features don't work, check `docker logs immich_machine_learning` on fileserver
- Verify network connectivity: `ping` and `curl` tests from mini PC
- Check Immich server logs: `docker logs immich_server` for ML connection errors

### Systemd Services for Split Deployment

**Mini PC** - Create standard systemd service for Immich:
```bash
sudo nano /etc/systemd/system/docker-compose-immich.service
```

Follow the same format as the media/web stacks (see earlier in this document).

**Fileserver** - The immich-ml container is deployed via Docker web UI and will auto-restart with Docker. If using Docker Compose instead, follow the same systemd service pattern. For web UI deployments, the restart policy (`unless-stopped`) handles automatic startup.
