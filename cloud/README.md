# Cloud Stack

Docker Compose configuration for self-hosted cloud services and collaboration tools.

## Services

### Nextcloud
**Ports:** 8080 (HTTP), 8443 (HTTPS)  
Complete self-hosted cloud solution with file storage, calendar, contacts, and extensive app ecosystem.

**Access:** `http://localhost:8080` or `https://localhost:8443`

### Immich
**Port:** 2283  
High-performance self-hosted photo and video backup solution, similar to Google Photos.

**Access:** `http://localhost:2283`

### Collabora Online
**Port:** 9980  
Online office suite that integrates with Nextcloud for document editing.

**Access:** Via Nextcloud integration (configure as Nextcloud app)

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
   - Database passwords (change from default!)
   - Collabora credentials
   - Nextcloud domain for Collabora integration

## Data Storage

Application data is stored in `/var/lib/containers/appdata/`:
- `/var/lib/containers/appdata/nextcloud/config/` - Nextcloud configuration
- `/var/lib/containers/appdata/nextcloud/data/` - Nextcloud user files
- `/var/lib/containers/appdata/nextcloud/db/` - Nextcloud database
- `/var/lib/containers/appdata/immich/upload/` - Immich uploaded photos
- `/var/lib/containers/appdata/immich/library/` - Immich processed library
- `/var/lib/containers/appdata/immich/db/` - Immich database

## Deployment

Start the cloud stack:
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

### Nextcloud
1. Navigate to `http://localhost:8080`
2. Create admin account on first access
3. Follow setup wizard
4. Configure database connection (should be pre-configured)
5. Install recommended apps
6. Configure trusted domains in config.php if needed

### Immich
1. Navigate to `http://localhost:2283`
2. Create admin account
3. Download mobile app from App Store or Google Play
4. Configure server URL in mobile app
5. Start uploading photos

### Collabora + Nextcloud Integration
1. In Nextcloud, go to Apps
2. Install "Nextcloud Office" app
3. Go to Settings → Nextcloud Office
4. Set Collabora server URL: `http://collabora:9980`
5. Test connection and save

## Database Management

### Nextcloud Database Backup
```bash
docker exec nextcloud-db mysqldump -u nextcloud -p nextcloud > nextcloud-backup.sql
```

### Immich Database Backup
```bash
docker exec immich-db pg_dump -U immich immich > immich-backup.sql
```

## Network

The cloud stack uses a custom bridge network `cloud_network` for service communication.

## Security Considerations

**Important:** This configuration is intended for local/internal use. For external access:

1. Use a reverse proxy (e.g., Traefik, Nginx Proxy Manager)
2. Enable SSL/TLS with valid certificates
3. Configure Nextcloud's `trusted_domains` and `overwrite.cli.url`
4. Use strong passwords for all services
5. Keep services updated regularly

## LinuxServer Images

Nextcloud uses LinuxServer.io image which provides:
- Consistent UID/GID handling via PUID/PGID
- Regular security updates
- Comprehensive documentation
- Active community support

Image format: `lscr.io/linuxserver/nextcloud:latest`

## Systemd Integration

To automatically start this stack on boot, create a systemd service:

```bash
sudo nano /etc/systemd/system/docker-compose-cloud.service
```

See [DEPLOYMENT.md](../DEPLOYMENT.md) for complete systemd configuration.

## Performance Tuning

### Nextcloud
- Enable Redis caching (requires additional container)
- Configure PHP memory limits
- Enable APCu for local caching
- Schedule background jobs via cron

### Immich
- Allocate sufficient resources for machine learning
- Configure hardware acceleration for thumbnails
- Regular database maintenance

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

### Nextcloud Maintenance Mode
```bash
# Enable
docker exec -u www-data nextcloud php occ maintenance:mode --on

# Disable
docker exec -u www-data nextcloud php occ maintenance:mode --off
```

### Run Nextcloud occ Commands
```bash
docker exec -u www-data nextcloud php occ [command]
```

## Troubleshooting

### Nextcloud Database Connection Issues
Check environment variables and ensure database container is running:
```bash
docker compose logs nextcloud-db
```

### Collabora Connection Failed
Verify:
1. Domain regex is correct in `.env`
2. Nextcloud can reach Collabora container
3. Check Collabora logs: `docker compose logs collabora`

### Immich Upload Issues
Check permissions on upload directory:
```bash
sudo chown -R 1000:1000 /var/lib/containers/appdata/immich
```

### Port Conflicts
If ports are already in use, modify them in `compose.yml` or use environment variables in `.env`.

## Backup Strategy

Regular backups are essential:

1. **Configuration**: Backup `compose.yml` and `.env`
2. **Application Data**: Backup `/var/lib/containers/appdata/`
3. **Databases**: Regular database dumps
4. **Test Restores**: Periodically test backup restoration

See [DEPLOYMENT.md](../DEPLOYMENT.md) for detailed backup procedures.
