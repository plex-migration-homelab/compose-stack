# Web Stack

Docker Compose configuration for web applications and dashboard services.

## Services

### Overseerr
**Port:** 5055  
Media request and discovery tool for Plex/Jellyfin. Allows users to request movies and TV shows.

**Access:** `http://localhost:5055`

### Wizarr
**Port:** 5690  
User invitation and management system for media servers. Simplifies onboarding new users to your Plex/Jellyfin server.

**Access:** `http://localhost:5690`

### Organizr
**Port:** 9983  
Unified dashboard for all your homelab services. Provides a single interface to access all your applications.

**Access:** `http://localhost:9983`

### Homepage
**Port:** 3000  
Modern, customizable application dashboard. Displays service status, system stats, and quick links.

**Access:** `http://localhost:3000`

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

## Data Storage

Application data is stored in `/var/lib/containers/appdata/`:
- `/var/lib/containers/appdata/overseerr/` - Overseerr configuration and database
- `/var/lib/containers/appdata/wizarr/` - Wizarr database
- `/var/lib/containers/appdata/organizr/` - Organizr configuration
- `/var/lib/containers/appdata/homepage/` - Homepage configuration

## Deployment

Start the web stack:
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

## Initial Setup

### Overseerr
1. Navigate to `http://localhost:5055`
2. Connect to your Plex or Jellyfin server
3. Configure Radarr/Sonarr integration
4. Set up user permissions

### Wizarr
1. Navigate to `http://localhost:5690`
2. Complete initial setup wizard
3. Configure invitation settings
4. Connect to media server

### Organizr
1. Navigate to `http://localhost:9983`
2. Create admin account
3. Add tabs for your services
4. Configure authentication

### Homepage
1. Navigate to `http://localhost:3000`
2. Edit configuration in `/var/lib/containers/appdata/homepage/`
3. Add services, bookmarks, and widgets
4. Customize layout and theme

## Network

The web stack uses a custom bridge network `web_network` for service communication.

## LinuxServer Images

Services using LinuxServer.io images (Overseerr, Organizr):
- Consistent UID/GID handling via PUID/PGID
- Regular security updates
- Comprehensive documentation
- Active community support

Images follow the format: `lscr.io/linuxserver/[service]:latest`

## Systemd Integration

To automatically start this stack on boot, create a systemd service:

```bash
sudo nano /etc/systemd/system/docker-compose-web.service
```

See [DEPLOYMENT.md](../DEPLOYMENT.md) for complete systemd configuration.

## Integration with Other Stacks

These web services are designed to integrate with your media stack:
- **Overseerr** → Radarr/Sonarr (media requests)
- **Wizarr** → Plex/Jellyfin (user management)
- **Organizr/Homepage** → All services (dashboard access)

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

## Troubleshooting

### Service won't start
Check logs:
```bash
docker compose logs [service-name]
```

### Permission issues
Verify PUID/PGID match your user:
```bash
id -u  # Check PUID
id -g  # Check PGID
```

### Port conflicts
If ports are already in use, modify them in `compose.yml` or use environment variables in `.env`.
