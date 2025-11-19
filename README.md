# Docker Compose Homelab Stack

This repository contains Docker Compose configurations for a complete homelab stack organized into three main categories:

- **Media Stack** (`media/`) - Media management and streaming services
- **Web Stack** (`web/`) - Web applications and dashboards
- **Cloud Stack** (`cloud/`) - Self-hosted cloud services and collaboration tools

## Structure

```
/srv/containers/
├── media/          # Media management stack
├── web/            # Web applications stack
└── cloud/          # Cloud services stack
```

Each directory contains:
- `compose.yml` - Docker Compose configuration
- `docker-compose.yml` - Symlink to compose.yml for compatibility
- `.env.example` - Example environment variables
- `README.md` - Stack-specific documentation

## Quick Start

1. Clone this repository to `/srv/containers/`
2. Navigate to the desired stack directory (media, web, or cloud)
3. Copy `.env.example` to `.env` and customize variables
4. Run `docker compose up -d` to start the stack

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
