# Docker Compose Homelab Stack

This repository contains Docker Compose configurations for a complete homelab stack organized into multiple specialized stacks:

- **Media Stack** (`media/`) - Media management and streaming services (Plex, Jellyfin)
- **Web Stack** (`web/`) - Web applications and dashboards (Nginx Proxy Manager, Overseerr, Wizarr)
- **Nextcloud Stack** (`nextcloud/`) - Self-hosted cloud storage and collaboration using Nextcloud AIO
- **Immich Stack** (`immich/`) - Self-hosted photo management and backup (main services)
- **Immich ML Stack** (`immich-ml/`) - Machine learning services for Immich with GPU acceleration

## Structure

```
/srv/containers/
├── media/          # Media streaming stack
├── web/            # Web applications stack
├── nextcloud/      # Nextcloud All-in-One stack
├── immich/         # Immich main services (mini PC)
└── immich-ml/      # Immich ML with GPU (fileserver)
```

Each directory contains:
- `compose.yml` - Docker Compose configuration
- `docker-compose.yml` - Symlink to compose.yml for compatibility
- `.env.example` - Example environment variables
- `README.md` - Stack-specific documentation

## Quick Start

1. Clone this repository to `/srv/containers/`
2. Navigate to the desired stack directory
3. Copy `.env.example` to `.env` and customize variables
4. Run `docker compose up -d` to start the stack

### Special Considerations

**Nextcloud AIO:**
- Uses a mastercontainer that manages other containers automatically
- Access setup interface at `http://YOUR_IP:8080` after first start
- See `nextcloud/README.md` for detailed setup instructions

**Immich Split Deployment:**
- Main stack (`immich/`) runs on mini PC - handles web UI, storage, database
- ML stack (`immich-ml/`) runs on fileserver - provides GPU-accelerated ML
- Configure `IMMICH_MACHINE_LEARNING_URL` in `immich/.env` to point to fileserver
- Requires NVIDIA Container Toolkit on fileserver
- See respective README files for detailed setup

## Environment Variables

All stacks use the following common environment variables:

- `PUID` - User ID for file permissions (default: 1000)
- `PGID` - Group ID for file permissions (default: 1000)
- `TZ` - Timezone (e.g., America/New_York)

## Data Storage

Application data is stored in `/var/lib/containers/appdata/` with subdirectories for each service.

## Deployment

See [DEPLOYMENT.md](DEPLOYMENT.md) for detailed deployment instructions, systemd integration, and management procedures.

## Maintenance

Each stack can be managed independently:

```bash
# Start a stack
cd /srv/containers/[stack-name]
docker compose up -d

# Stop a stack
docker compose down

# View logs
docker compose logs -f

# Update containers
docker compose pull
docker compose up -d
```
