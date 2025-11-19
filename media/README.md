# Media Stack

Docker Compose configuration for media management and streaming services.

## Services

This stack is designed to host media management services such as:
- **Plex/Jellyfin** - Media server
- **Radarr** - Movie management
- **Sonarr** - TV show management
- **Lidarr** - Music management
- **Prowlarr** - Indexer management
- **qBittorrent/Transmission** - Download client

## Configuration

1. Copy the example environment file:
   ```bash
   cp .env.example .env
   ```

2. Edit `.env` with your configuration:
   ```bash
   nano .env
   ```

3. Configure the following variables:
   - `PUID` - Your user ID (run `id -u` to find)
   - `PGID` - Your group ID (run `id -g` to find)
   - `TZ` - Your timezone
   - Media paths for your library

## Data Storage

Application data is stored in `/var/lib/containers/appdata/` with separate directories for each service:
- `/var/lib/containers/appdata/plex/` - Plex configuration
- `/var/lib/containers/appdata/radarr/` - Radarr configuration
- `/var/lib/containers/appdata/sonarr/` - Sonarr configuration
- etc.

## Deployment

Start the media stack:
```bash
docker compose up -d
```

Stop the stack:
```bash
docker compose down
```

View logs:
```bash
docker compose logs -f
```

## Network

The media stack uses a custom bridge network `media_network` for service communication.

## LinuxServer Images

This stack uses LinuxServer.io Docker images which provide:
- Consistent UID/GID handling via PUID/PGID
- Regular security updates
- Comprehensive documentation
- Active community support

All images follow the format: `lscr.io/linuxserver/[service]:latest`

## Systemd Integration

To automatically start this stack on boot, create a systemd service:

```bash
sudo nano /etc/systemd/system/docker-compose-media.service
```

See [DEPLOYMENT.md](../DEPLOYMENT.md) for complete systemd configuration.

## Accessing Services

Once running, services are typically available at:
- Plex: `http://localhost:32400/web`
- Radarr: `http://localhost:7878`
- Sonarr: `http://localhost:8989`

(Adjust ports based on your configuration)

## Maintenance

### Update containers
```bash
docker compose pull
docker compose up -d
```

### Restart a specific service
```bash
docker compose restart [service-name]
```

### View service logs
```bash
docker compose logs -f [service-name]
```
