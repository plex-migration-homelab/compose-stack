# Quick Reference Guide

Fast reference for common commands and configurations.

## 📖 Documentation Quick Links

- [README.md](README.md) - Main documentation and overview
- [DEPLOYMENT.md](DEPLOYMENT.md) - Deployment and setup guide
- [ARCHITECTURE.md](ARCHITECTURE.md) - System architecture details
- [FAQ.md](FAQ.md) - Frequently asked questions
- [CONTRIBUTING.md](CONTRIBUTING.md) - How to contribute
- [SECURITY.md](SECURITY.md) - Security guidelines
- [CHANGELOG.md](CHANGELOG.md) - Version history

## 🚀 Common Commands

### Service Management

```bash
# Start a stack
cd /srv/containers/[stack-name]
docker compose up -d

# Stop a stack
docker compose down

# Restart a service
docker compose restart [service-name]

# View status
docker compose ps

# View logs
docker compose logs -f [service-name]

# Update containers
docker compose pull
docker compose up -d
```

### System Commands

```bash
# View all running containers
docker ps

# View all containers (including stopped)
docker ps -a

# Resource usage
docker stats

# Clean up unused resources
docker system prune
docker image prune -f

# Check disk usage
docker system df
```

### Systemd Management

```bash
# Start service
sudo systemctl start docker-compose-web.service

# Stop service
sudo systemctl stop docker-compose-web.service

# Restart service
sudo systemctl restart docker-compose-web.service

# Check status
sudo systemctl status docker-compose-web.service

# View logs
journalctl -u docker-compose-web.service -f

# Enable on boot
sudo systemctl enable docker-compose-web.service
```

## 🔧 Configuration

### Environment Variables

```bash
# Common variables for all stacks
PUID=1000          # User ID (run: id -u)
PGID=1000          # Group ID (run: id -g)
TZ=America/New_York # Timezone

# Find your PUID/PGID
id -u  # PUID
id -g  # PGID
```

### Port Reference

#### Web Stack
- Overseerr: `5055`
- Wizarr: `5690`
- Organizr: `9983`
- Homepage: `3000`

#### Cloud Stack
- Nextcloud HTTP: `8080`
- Nextcloud HTTPS: `8443`
- Immich: `2283`
- Collabora: `9980`

#### Media Stack
- Plex: `32400` (example)
- Radarr: `7878` (example)
- Sonarr: `8989` (example)
- Prowlarr: `9696` (example)

## 🛠️ Troubleshooting

### Check Service Logs

```bash
# Recent logs
docker compose logs --tail=50 [service-name]

# Follow logs in real-time
docker compose logs -f [service-name]

# Logs with timestamps
docker compose logs -f --timestamps [service-name]

# Save logs to file
docker compose logs [service-name] > service.log
```

### Fix Permission Issues

```bash
# Fix appdata permissions
sudo chown -R $(id -u):$(id -g) /var/lib/containers/appdata/[service]

# Verify permissions
ls -la /var/lib/containers/appdata/[service]
```

### Port Conflicts

```bash
# Find what's using a port
sudo lsof -i :PORT_NUMBER
sudo ss -tulpn | grep PORT_NUMBER

# Kill process on port
sudo kill -9 $(sudo lsof -t -i:PORT_NUMBER)
```

### Restart Docker

```bash
# Restart Docker daemon
sudo systemctl restart docker

# Check Docker status
sudo systemctl status docker

# View Docker logs
journalctl -u docker.service -n 50
```

### Container Issues

```bash
# Recreate container
docker compose up -d --force-recreate [service-name]

# Remove and recreate
docker compose rm -f [service-name]
docker compose up -d [service-name]

# Inspect container
docker inspect [container-name]

# Execute command in container
docker exec -it [container-name] /bin/bash
```

## 💾 Backup & Restore

### Quick Backup

```bash
# Backup configurations
cd /srv/containers
tar -czf ~/config-backup-$(date +%F).tar.gz \
  --exclude='.git' media/ web/ cloud/ *.md

# Backup application data
sudo tar -czf ~/appdata-backup-$(date +%F).tar.gz \
  -C /var/lib/containers appdata/

# Backup specific service
sudo tar -czf ~/nextcloud-backup-$(date +%F).tar.gz \
  -C /var/lib/containers/appdata nextcloud/
```

### Quick Restore

```bash
# Restore configurations
sudo tar -xzf ~/config-backup-*.tar.gz -C /srv/containers

# Restore application data
sudo tar -xzf ~/appdata-backup-*.tar.gz -C /var/lib/containers

# Fix permissions
sudo chown -R $(id -u):$(id -g) /var/lib/containers/appdata
```

### Database Backup

```bash
# Nextcloud database
docker exec nextcloud-db mysqldump -u nextcloud -p nextcloud > nextcloud.sql

# Immich database
docker exec immich-db pg_dump -U immich immich > immich.sql
```

## 🔄 Updates

### Update Single Stack

```bash
cd /srv/containers/[stack-name]
docker compose pull
docker compose up -d
docker image prune -f
```

### Update All Stacks

```bash
cd /srv/containers
for stack in media web cloud; do
  cd $stack && docker compose pull && docker compose up -d && cd ..
done
docker image prune -f
```

### Update Docker

```bash
sudo apt update
sudo apt upgrade docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo systemctl restart docker
```

## 📊 Monitoring

### Resource Monitoring

```bash
# Real-time resource usage
docker stats

# Specific container stats
docker stats [container-name]

# Disk usage
docker system df

# Detailed disk usage
docker system df -v
```

### Health Checks

```bash
# Check all containers in stack
docker compose ps

# Check specific service
docker compose ps [service-name]

# Check container health
docker inspect --format='{{.State.Health.Status}}' [container-name]
```

### Log Monitoring

```bash
# System logs
journalctl -f

# Docker daemon logs
journalctl -u docker.service -f

# Service logs
journalctl -u docker-compose-web.service -f
```

## 🔒 Security Quick Checks

### Security Checklist

```bash
# Check for running containers
docker ps

# Check exposed ports
sudo netstat -tulpn | grep LISTEN

# Check firewall status
sudo ufw status

# Check for updates
docker images | grep -v REPOSITORY | awk '{print $1":"$2}' | xargs -L1 docker pull

# Scan for vulnerabilities (if docker scan available)
docker scan [image-name]
```

### Password Management

```bash
# Generate secure password
openssl rand -base64 32

# Generate multiple passwords
for i in {1..5}; do openssl rand -base64 32; done
```

## 📁 File Locations

### Configuration Files

```
/srv/containers/               # Stack configurations
├── media/compose.yml         # Media stack config
├── web/compose.yml           # Web stack config
├── cloud/compose.yml         # Cloud stack config
├── media/.env               # Media environment vars
├── web/.env                 # Web environment vars
└── cloud/.env               # Cloud environment vars
```

### Data Directories

```
/var/lib/containers/appdata/  # Application data
├── overseerr/               # Overseerr data
├── nextcloud/               # Nextcloud data
├── immich/                  # Immich data
└── [other services]/        # Service-specific data
```

### System Files

```
/etc/systemd/system/          # Systemd service files
├── docker-compose-media.service
├── docker-compose-web.service
└── docker-compose-cloud.service

/var/log/                     # Log files
└── syslog                    # System logs
```

## 🌐 Access URLs

Once services are running, access them at:

```
# Web Stack
http://localhost:5055         # Overseerr
http://localhost:5690         # Wizarr
http://localhost:9983         # Organizr
http://localhost:3000         # Homepage

# Cloud Stack
http://localhost:8080         # Nextcloud
http://localhost:2283         # Immich

# Replace localhost with server IP for remote access
http://192.168.1.100:5055    # Example
```

## 🆘 Emergency Commands

### Service Not Responding

```bash
# Force restart
docker compose restart [service-name]

# Nuclear option: recreate everything
docker compose down
docker compose up -d --force-recreate
```

### Disk Full

```bash
# Clean up Docker resources
docker system prune -a --volumes

# Check what's using space
du -sh /var/lib/containers/appdata/*
du -sh /var/lib/docker/*
```

### Network Issues

```bash
# Restart Docker network
docker compose down
docker network prune
docker compose up -d
```

### Complete Reset (DANGER!)

```bash
# Stop all containers
docker stop $(docker ps -aq)

# Remove all containers
docker rm $(docker ps -aq)

# Remove all networks
docker network prune -f

# Remove all volumes (DELETES DATA!)
docker volume prune -f

# Remove all images
docker rmi $(docker images -q)

# Start fresh
cd /srv/containers/[stack-name]
docker compose up -d
```

## 📚 Additional Resources

### Official Documentation

- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [LinuxServer.io Documentation](https://docs.linuxserver.io/)

### Service Documentation

- [Nextcloud Documentation](https://docs.nextcloud.com/)
- [Immich Documentation](https://immich.app/docs)
- [Overseerr Documentation](https://docs.overseerr.dev/)
- [Homepage Documentation](https://gethomepage.dev/)

### Community Resources

- [Docker Community Forums](https://forums.docker.com/)
- [Reddit: r/selfhosted](https://reddit.com/r/selfhosted)
- [LinuxServer.io Discord](https://discord.gg/YWrKVTn)

## 💡 Tips & Tricks

### Aliases

Add to `~/.bashrc` or `~/.zshrc`:

```bash
# Docker aliases
alias dps='docker ps'
alias dlog='docker compose logs -f'
alias dup='docker compose up -d'
alias ddown='docker compose down'
alias dstats='docker stats'

# Stack aliases
alias webstack='cd /srv/containers/web'
alias cloudstack='cd /srv/containers/cloud'
alias mediastack='cd /srv/containers/media'
```

### Useful One-Liners

```bash
# Stop all running containers
docker stop $(docker ps -q)

# Remove all stopped containers
docker rm $(docker ps -a -q)

# Remove dangling images
docker rmi $(docker images -f "dangling=true" -q)

# View all container IPs
docker ps -q | xargs -n 1 docker inspect --format '{{.Name}} {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'

# Follow logs from all containers
docker compose logs -f --tail=10
```

---

**Quick Tip**: Bookmark this page for fast reference! Press `Ctrl+F` to search for specific commands.

For detailed explanations, refer to the full documentation files listed at the top.
