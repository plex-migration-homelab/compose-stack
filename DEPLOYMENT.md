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
sudo systemctl enable docker-compose-cloud.service

# Start services
sudo systemctl start docker-compose-media.service
sudo systemctl start docker-compose-web.service
sudo systemctl start docker-compose-cloud.service

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

## Advanced Configuration

### Reverse Proxy Setup

For secure external access, use a reverse proxy. Example with Nginx:

```nginx
# /etc/nginx/sites-available/overseerr
server {
    listen 80;
    server_name requests.yourdomain.com;
    
    location / {
        proxy_pass http://localhost:5055;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable SSL with Let's Encrypt:

```bash
sudo certbot --nginx -d requests.yourdomain.com
```

### Resource Limits

Limit container resources in `compose.yml`:

```yaml
services:
  service-name:
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          memory: 1G
```

### Healthchecks

Add healthchecks to monitor service status:

```yaml
services:
  service-name:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

### Logging Configuration

Configure logging driver:

```yaml
services:
  service-name:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

## Performance Optimization

### Docker Storage Driver

Check and optimize storage driver:

```bash
# Check current driver
docker info | grep "Storage Driver"

# For best performance, use overlay2
# Edit /etc/docker/daemon.json
{
  "storage-driver": "overlay2"
}
```

### Disk Performance

Use SSD for appdata:

```bash
# Mount SSD to appdata location
sudo mount /dev/sdX /var/lib/containers/appdata

# Add to /etc/fstab for persistence
/dev/sdX /var/lib/containers/appdata ext4 defaults 0 2
```

### Network Performance

Optimize Docker network:

```bash
# Edit /etc/docker/daemon.json
{
  "bip": "172.26.0.1/16",
  "mtu": 1500
}
```

### Memory Management

Configure Docker to use appropriate resources:

```bash
# Edit /etc/docker/daemon.json
{
  "default-shm-size": "256M",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

## Automated Maintenance

### Auto-Update Script

Create a script for automatic updates:

```bash
#!/bin/bash
# /usr/local/bin/update-homelab.sh

STACKS=("media" "web" "cloud")
BASE_DIR="/srv/containers"

for stack in "${STACKS[@]}"; do
    echo "Updating $stack stack..."
    cd "$BASE_DIR/$stack"
    docker compose pull
    docker compose up -d
done

# Cleanup
docker image prune -f

echo "Update complete!"
```

Make executable:

```bash
sudo chmod +x /usr/local/bin/update-homelab.sh
```

### Cron Jobs

Schedule automatic updates:

```bash
# Edit crontab
sudo crontab -e

# Add weekly updates (Sundays at 3 AM)
0 3 * * 0 /usr/local/bin/update-homelab.sh >> /var/log/homelab-update.log 2>&1
```

### Backup Automation

Automate backups:

```bash
#!/bin/bash
# /usr/local/bin/backup-homelab.sh

BACKUP_DIR="/path/to/backups"
DATE=$(date +%F)

# Backup configurations
cd /srv/containers
tar -czf "$BACKUP_DIR/config-$DATE.tar.gz" \
  --exclude='.git' \
  media/ web/ cloud/ *.md

# Backup appdata
tar -czf "$BACKUP_DIR/appdata-$DATE.tar.gz" \
  -C /var/lib/containers appdata/

# Keep only last 7 days
find "$BACKUP_DIR" -name "*.tar.gz" -mtime +7 -delete

echo "Backup completed: $DATE"
```

Schedule daily backups:

```bash
# Add to crontab
0 2 * * * /usr/local/bin/backup-homelab.sh >> /var/log/homelab-backup.log 2>&1
```

## Disaster Recovery

### Recovery Checklist

1. **Install fresh system**
   - Install OS
   - Install Docker and Docker Compose
   - Configure network

2. **Restore configurations**
   ```bash
   sudo mkdir -p /srv/containers
   sudo tar -xzf config-backup.tar.gz -C /srv/containers
   sudo chown -R $USER:$USER /srv/containers
   ```

3. **Restore application data**
   ```bash
   sudo mkdir -p /var/lib/containers
   sudo tar -xzf appdata-backup.tar.gz -C /var/lib/containers
   sudo chown -R $(id -u):$(id -g) /var/lib/containers/appdata
   ```

4. **Restore environment files**
   ```bash
   # Copy .env files from secure backup location
   cp .env.backup /srv/containers/web/.env
   cp .env.backup /srv/containers/cloud/.env
   ```

5. **Start services**
   ```bash
   cd /srv/containers/web
   docker compose up -d
   ```

6. **Verify functionality**
   - Check service accessibility
   - Verify data integrity
   - Test core functionality

### Testing Recovery

Regularly test your recovery process:

1. Set up a test VM
2. Attempt full restoration
3. Verify all services work
4. Document any issues or missing steps
5. Update recovery procedures

## Advanced Troubleshooting

### Debug Mode

Enable debug logging:

```bash
# Edit /etc/docker/daemon.json
{
  "debug": true,
  "log-level": "debug"
}

# Restart Docker
sudo systemctl restart docker
```

### Container Inspection

Deep dive into container issues:

```bash
# Inspect container configuration
docker inspect [container-name]

# Check container processes
docker top [container-name]

# Execute commands in container
docker exec -it [container-name] /bin/bash

# Check container resource usage
docker stats [container-name]
```

### Network Debugging

Troubleshoot network issues:

```bash
# List all networks
docker network ls

# Inspect network
docker network inspect [network-name]

# Check network connectivity between containers
docker exec [container1] ping [container2]

# Check DNS resolution
docker exec [container-name] nslookup [service-name]
```

### Database Issues

Database-specific troubleshooting:

```bash
# Nextcloud database
docker exec -it nextcloud-db mysql -u nextcloud -p
SHOW DATABASES;
USE nextcloud;
SHOW TABLES;

# Immich database
docker exec -it immich-db psql -U immich -d immich
\l  # List databases
\dt # List tables
```

### Log Analysis

Comprehensive log checking:

```bash
# Docker daemon logs
journalctl -u docker.service -n 100 --no-pager

# Service logs with timestamps
docker compose logs -f --timestamps [service-name]

# All logs for a stack
docker compose logs --tail=100

# Save logs to file
docker compose logs > service-logs.txt
```

## Migration Guide

### Moving to a New Server

1. **Prepare new server**
   - Install Docker and Docker Compose
   - Configure network and storage
   - Create necessary directories

2. **Stop services on old server**
   ```bash
   cd /srv/containers/[stack-name]
   docker compose down
   ```

3. **Transfer data**
   ```bash
   # On old server
   rsync -avz /srv/containers/ newserver:/srv/containers/
   rsync -avz /var/lib/containers/appdata/ newserver:/var/lib/containers/appdata/
   ```

4. **Start services on new server**
   ```bash
   cd /srv/containers/[stack-name]
   docker compose up -d
   ```

5. **Update DNS/networking**
   - Point domain names to new IP
   - Update firewall rules
   - Test accessibility

### Upgrading Docker

```bash
# Check current version
docker --version

# Update Docker
sudo apt update
sudo apt upgrade docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Restart services
sudo systemctl restart docker

# Verify
docker --version
docker compose version
```

## Conclusion

This deployment guide covers everything from basic installation to advanced troubleshooting. For additional help:

- Review [FAQ.md](FAQ.md) for common questions
- Check [ARCHITECTURE.md](ARCHITECTURE.md) for design details
- Consult service-specific READMEs for detailed configurations
- Open an issue on GitHub for specific problems

Remember to:
- Keep your system updated
- Backup regularly
- Test your backups
- Monitor resource usage
- Review logs periodically
- Keep security in mind
