# Frequently Asked Questions (FAQ)

## General Questions

### What is this project?

This is a collection of Docker Compose configurations for running a complete homelab stack. It includes media management, web applications, and cloud services, organized into three main categories for easy deployment and management.

### Do I need to deploy all stacks?

No! Each stack (media, web, cloud) is independent. Deploy only the services you need.

### What are the system requirements?

**Minimum:**
- Docker Engine 24.0+
- Docker Compose v2.20+
- Linux system with systemd
- 2GB RAM (varies by services deployed)
- 20GB storage (plus space for your data)

**Recommended:**
- 4+ CPU cores
- 8GB+ RAM
- SSD for application data
- Sufficient storage for media/files

## Installation Questions

### Where should I install this?

The recommended installation path is `/srv/containers/`, but you can use any location. Just update the paths in your configuration accordingly.

### Can I run this on Windows or macOS?

These configurations are designed for Linux with Docker. While Docker Desktop on Windows/macOS might work, paths and permissions may need adjustment. Linux is strongly recommended.

### Do I need root access?

You need root/sudo access for:
- Initial Docker installation
- Creating system directories (`/srv`, `/var/lib/containers`)
- Setting up systemd services

Normal operation can run as a regular user if properly configured.

## Configuration Questions

### What should PUID and PGID be?

Set them to your user's UID and GID. Find yours with:
```bash
id -u  # Returns PUID
id -g  # Returns PGID
```

Typically `1000` for the first user on a system.

### How do I change default ports?

Edit the port mappings in `compose.yml` or add port variables to your `.env` file. Format: `"HOST_PORT:CONTAINER_PORT"`

### Can I use a different data directory?

Yes! Update the volume paths in `compose.yml` to point to your preferred location. Ensure proper permissions.

### Should I change the default passwords?

**YES!** Always change default passwords in `.env`, especially for:
- Database passwords
- Collabora credentials
- Any admin accounts

## Service-Specific Questions

### Which media server should I use?

- **Plex**: More polished, better apps, requires Plex Pass for some features
- **Jellyfin**: Fully open-source, no paid features, active development

Both work great! Add the one you prefer to `media/compose.yml`.

### How do I add a new service?

1. Edit the appropriate stack's `compose.yml`
2. Add the service configuration
3. Update the stack's README.md
4. Add any required environment variables to `.env.example`
5. Deploy with `docker compose up -d`

### Can services communicate across stacks?

Yes, but they're on separate networks by default. To enable cross-stack communication:
- Use `external_links` in compose.yml
- Put services on the same network
- Use host networking (less secure)

## Troubleshooting Questions

### My service won't start. What should I check?

1. **Check logs**: `docker compose logs [service-name]`
2. **Verify permissions**: Ensure PUID/PGID are correct
3. **Check ports**: Make sure ports aren't already in use
4. **Validate .env**: Ensure all required variables are set
5. **Check disk space**: Ensure sufficient storage available

### How do I fix permission denied errors?

```bash
# Fix appdata permissions
sudo chown -R $PUID:$PGID /var/lib/containers/appdata/[service]

# Verify ownership
ls -la /var/lib/containers/appdata/[service]
```

### Services show as "unhealthy" or constantly restarting?

1. Check service dependencies are running (e.g., databases)
2. Review logs for error messages
3. Verify environment variables are correct
4. Check if configuration files are corrupted
5. Try recreating: `docker compose up -d --force-recreate [service]`

### How do I resolve port conflicts?

```bash
# Find what's using a port
sudo lsof -i :PORT_NUMBER

# Or with ss
sudo ss -tulpn | grep PORT_NUMBER
```

Then either stop the conflicting service or change the port in `compose.yml`.

## Networking Questions

### How do I access services from other devices?

Services bind to `localhost` by default. To access from other devices:
1. Bind to `0.0.0.0` instead of `127.0.0.1`
2. Configure your firewall to allow the ports
3. Access via `http://SERVER_IP:PORT`

### Should I expose services to the internet?

**Not directly!** Use a reverse proxy with SSL/TLS:
- Traefik
- Nginx Proxy Manager
- Caddy

See DEPLOYMENT.md for security considerations.

### What about VPNs?

For secure remote access, consider:
- WireGuard
- Tailscale
- Setting up a VPN container in your stack

## Maintenance Questions

### How often should I update?

Recommended schedule:
- **Security updates**: Immediately
- **Routine updates**: Monthly
- **Major versions**: When stable, after reviewing changelogs

### How do I update containers?

```bash
cd /srv/containers/[stack-name]
docker compose pull
docker compose up -d
```

Docker Compose will only restart containers with new images.

### How do I backup my data?

**Essential backups:**
1. Compose configurations and `.env` files
2. `/var/lib/containers/appdata/` (application data)
3. Database dumps for services using databases

See DEPLOYMENT.md for detailed backup procedures.

### How do I migrate to a new server?

1. Backup all data (configs and appdata)
2. Install Docker on new server
3. Restore configurations to `/srv/containers`
4. Restore appdata to `/var/lib/containers/appdata`
5. Fix ownership: `sudo chown -R $PUID:$PGID /var/lib/containers/appdata`
6. Start services: `docker compose up -d`

## Performance Questions

### My services are slow. How can I improve performance?

**General tips:**
- Use SSD for appdata storage
- Allocate sufficient RAM to Docker
- Enable hardware acceleration where supported
- Use caching (Redis for Nextcloud, etc.)
- Monitor resource usage: `docker stats`

### How much disk space do I need?

**Per stack (excluding your media/data):**
- Media stack: 5-10GB
- Web stack: 2-5GB
- Cloud stack: 10-20GB

Plus storage for your actual media, photos, files, etc.

## Docker Questions

### What's the difference between `docker compose` and `docker-compose`?

- `docker compose` (v2): Built into Docker CLI, recommended
- `docker-compose` (v1): Standalone Python tool, deprecated

Always use `docker compose` (v2).

### Can I use Podman instead of Docker?

Potentially, but these configs are optimized for Docker. You may need adjustments for Podman compatibility.

### How do I clean up unused Docker resources?

```bash
# Remove unused containers, networks, images
docker system prune

# Include volumes (BE CAREFUL!)
docker system prune --volumes

# Check disk usage
docker system df
```

## Still Have Questions?

- Check the [README.md](README.md) and [DEPLOYMENT.md](DEPLOYMENT.md)
- Review service-specific READMEs in each stack directory
- Search existing [GitHub Issues](https://github.com/plex-migration-homelab/compose-stack/issues)
- Open a new issue with the "question" label
