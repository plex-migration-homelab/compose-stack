# Nextcloud All-in-One Stack

This stack deploys Nextcloud using the official All-in-One (AIO) container, which includes all necessary components in a single, easy-to-manage deployment.

## What's Included

Nextcloud AIO automatically manages these components:
- **Nextcloud** - File sync and share platform
- **PostgreSQL** - Database
- **Redis** - Caching
- **Apache** - Web server
- **Collabora** - Online document editing
- **Talk** - Video conferencing
- **Imaginary** - Image processing
- **Fulltextsearch** - Search functionality
- **Automatic Backups** - Built-in backup system

## Quick Start

1. Copy the environment file:
   ```bash
   cp .env.example .env
   nano .env  # Adjust settings if needed
   ```

2. Start the stack:
   ```bash
   docker compose up -d
   ```

3. Access the AIO admin interface:
   ```
   http://YOUR_SERVER_IP:8080
   ```

4. Follow the setup wizard to:
   - Set your domain name
   - Configure SSL certificates (recommended: Let's Encrypt)
   - Choose which optional components to enable
   - Set up admin account

## Important Notes

### First-Time Setup
- The AIO mastercontainer will display a one-time setup password
- Access it via: `docker logs nextcloud-aio-mastercontainer`
- Use this password to log into the AIO interface

### Storage
- By default, data is stored in Docker volumes
- For production, consider mounting external storage by setting `NEXTCLOUD_DATADIR` in `.env`
- Example: `NEXTCLOUD_DATADIR=/mnt/nas-storage/nextcloud`

### Networking
- Port 8080: AIO admin interface
- Port 443: Nextcloud HTTPS access (managed by AIO)
- AIO handles its own reverse proxy internally

### Reverse Proxy
If using an external reverse proxy (like Nginx Proxy Manager):
- Configure it to pass through to the Nextcloud container
- Ensure proper headers are set for WebDAV and CalDAV
- See: https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md

### Updates
AIO handles updates automatically through the admin interface. You can also update the mastercontainer:
```bash
docker compose pull
docker compose up -d
```

### Backups
- AIO includes built-in backup functionality
- Configure backup location in the AIO interface
- Backups include database, files, and configuration
- Recommended: Store backups on separate storage

## Accessing Nextcloud

After setup, access Nextcloud at:
- Via domain: `https://your-domain.com`
- Via IP (not recommended for production): `https://YOUR_SERVER_IP:443`

## Troubleshooting

### View Logs
```bash
docker logs nextcloud-aio-mastercontainer
docker compose logs -f
```

### Reset AIO
If you need to start over:
```bash
docker compose down -v
docker compose up -d
```

### Check Container Status
```bash
docker ps | grep nextcloud
```

## Resources

- [Nextcloud AIO Documentation](https://github.com/nextcloud/all-in-one)
- [Nextcloud Documentation](https://docs.nextcloud.com/)
- [AIO FAQ](https://github.com/nextcloud/all-in-one#faq)
