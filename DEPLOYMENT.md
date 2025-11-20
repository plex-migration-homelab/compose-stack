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

Repeat for web, cloud, and management stacks, changing the description and working directory.

### Enable and Start Services

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable services to start on boot
sudo systemctl enable docker-compose-management.service  # Start first for monitoring
sudo systemctl enable docker-compose-media.service
sudo systemctl enable docker-compose-web.service
sudo systemctl enable docker-compose-cloud.service

# Start services
sudo systemctl start docker-compose-management.service
sudo systemctl start docker-compose-media.service
sudo systemctl start docker-compose-web.service
sudo systemctl start docker-compose-cloud.service

# Check status
sudo systemctl status docker-compose-management.service
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

### Portainer Web UI

The management stack includes Portainer for GUI-based container management:

1. **Access Portainer**: Navigate to `http://YOUR_SERVER_IP:9000` or `https://YOUR_SERVER_IP:9443`
2. **Initial Setup**: Create an admin account on first access
3. **Connect Environment**: Portainer auto-detects the local Docker environment
4. **View Stacks**: All your compose stacks (media, web, cloud, management) will appear under "Stacks"

Portainer provides:
- Real-time container stats and logs
- Quick restart/stop/start operations
- Compose file editing through UI
- Resource usage monitoring
- Network and volume management

**Note**: Portainer works alongside your CLI workflow - changes made in either interface are reflected in both.
