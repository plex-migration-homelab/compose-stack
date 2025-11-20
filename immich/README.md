# Immich Main Stack (Mini PC)

This stack runs the main Immich photo management services on the mini PC. The machine learning component runs separately on the fileserver with GPU acceleration.

## What's Included

- **Immich Server** - Web interface and API
- **Immich Microservices** - Background job processing
- **PostgreSQL** - Database with vector search support
- **Redis** - Cache and job queue

## Architecture

This is a **split deployment** where:
- **Mini PC** (this stack) - Runs the main services, web UI, and database
- **Fileserver** (immich-ml stack) - Runs GPU-accelerated ML for facial recognition, object detection, etc.

## Prerequisites

1. The fileserver must be running the `immich-ml` stack
2. Network connectivity between mini PC and fileserver
3. Sufficient storage for photos/videos

## Quick Start

1. Copy and configure environment:
   ```bash
   cp .env.example .env
   nano .env
   ```

2. **Important:** Set the correct ML URL in `.env`:
   ```bash
   IMMICH_MACHINE_LEARNING_URL=http://YOUR_FILESERVER_IP:3003
   ```

3. Create storage directories:
   ```bash
   sudo mkdir -p /var/lib/containers/appdata/immich/{upload,postgres}
   sudo chown -R 1000:1000 /var/lib/containers/appdata/immich
   ```

4. Start the stack:
   ```bash
   docker compose up -d
   ```

5. Access Immich:
   ```
   http://YOUR_MINI_PC_IP:2283
   ```

6. Create your admin account on first access

## Configuration

### Storage Locations

- **UPLOAD_LOCATION**: Where photos/videos are stored
  - Recommended: Fast local SSD
  - Alternative: NFS mount for network storage
  - Ensure adequate space (photos can be large!)

- **DB_DATA_LOCATION**: PostgreSQL database storage
  - Recommended: Reliable local storage
  - Keep on SSD for performance

### Machine Learning Connection

The `IMMICH_MACHINE_LEARNING_URL` must point to the fileserver running the ML container:
```bash
# Format
IMMICH_MACHINE_LEARNING_URL=http://FILESERVER_IP:3003

# Example
IMMICH_MACHINE_LEARNING_URL=http://192.168.1.100:3003
```

**Verify connectivity:**
```bash
curl http://FILESERVER_IP:3003/ping
```

### Database Security

Change the default database password in `.env`:
```bash
DB_PASSWORD=your_strong_password_here
```

## Mobile Apps

Immich has mobile apps for iOS and Android:
- **iOS**: Available on App Store
- **Android**: Available on Google Play or F-Droid

Configure the app to connect to: `http://YOUR_MINI_PC_IP:2283`

## Reverse Proxy Setup

For external access, configure your reverse proxy (Nginx Proxy Manager) to:
1. Proxy pass to `http://mini-pc-ip:2283`
2. Set proper headers for WebSocket support:
   ```
   proxy_set_header Upgrade $http_upgrade;
   proxy_set_header Connection "upgrade";
   ```

## Maintenance

### View Logs
```bash
docker compose logs -f
docker compose logs -f immich-server
```

### Backup

Critical data to backup:
1. **Photos/Videos**: `${UPLOAD_LOCATION}`
2. **Database**: `${DB_DATA_LOCATION}`
3. **Configuration**: `.env` file

Example backup:
```bash
# Stop services
docker compose down

# Backup
sudo tar -czf immich-backup-$(date +%F).tar.gz \
  /var/lib/containers/appdata/immich

# Restart services
docker compose up -d
```

### Updates

```bash
docker compose pull
docker compose up -d
```

### Reset/Fresh Install

```bash
docker compose down -v
# Delete data directories if starting completely fresh
docker compose up -d
```

## Troubleshooting

### ML Connection Issues

If facial recognition isn't working:
1. Check ML container on fileserver: `ssh fileserver "docker ps | grep immich"`
2. Test connectivity: `curl http://FILESERVER_IP:3003/ping`
3. Check firewall rules on fileserver
4. Verify `IMMICH_MACHINE_LEARNING_URL` in `.env`

### Performance Issues

- Ensure database is on SSD
- Monitor disk space for uploads
- Check network bandwidth if using NFS
- Review logs: `docker compose logs -f`

### Database Issues

Reset database (WARNING: deletes all data):
```bash
docker compose down
sudo rm -rf /var/lib/containers/appdata/immich/postgres
docker compose up -d
```

## Resources

- [Immich Documentation](https://immich.app/docs)
- [Immich GitHub](https://github.com/immich-app/immich)
- [Immich Discord](https://discord.gg/immich)
