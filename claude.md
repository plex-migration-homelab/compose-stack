# Claude Technical Reference: Docker Compose Homelab

Technical reference for providing support on this UBlue CoreOS-based homelab Docker Compose stack.

## System Architecture

### Hardware Environment
- **NAB9 Mini PC** (Primary host)
  - CPU: 12th gen Intel i5 with QuickSync
  - RAM: 32GB
  - OS: UBlue CoreOS (immutable, rpm-ostree based)
  - Runtime: Docker (not Podman)
  - Storage: Local appdata + NFS mounts

- **Unraid File Server**
  - ZFS array with NFS exports
  - Hosts Immich-ML service (NVIDIA P2000 GPU)
  - Media storage: `/mnt/nas/media`

- **Digital Ocean VPS**
  - Runs NginxProxyManager
  - WireGuard tunnel to mini PC
  - Public-facing service endpoint

### Network Topology
- **WireGuard Tunnel**: VPS ↔ Mini PC for secure public access
- **Tailscale**: Internal network access across all devices
- **NFS**: File server → Mini PC for media library

### Critical Constraint
**Remote hosting**: Mini PC is physically remote. Boot failures or systemd issues require Marcus intervention. Stability is paramount.

## Repository Structure

```
/srv/containers/
├── management/
│   └── docker-compose.yml       # Note: NOT compose.yml
├── media/
│   ├── compose.yml
│   └── docker-compose.yml       # Symlink to compose.yml
├── web/
│   ├── compose.yml
│   └── docker-compose.yml       # Symlink to compose.yml
├── cloud/
│   ├── nextcloud/               # Nextcloud AIO (all-in-one)
│   ├── immich/                  # Immich main services (mini PC)
│   └── immich-ml/               # Documentation only (service on Unraid)
├── DEPLOYMENT.md
├── README.md
├── claude.md                    # This file
└── agents.md                    # AI agent operational guidelines
```

### File Naming Convention
- **Management stack**: Uses single `docker-compose.yml` (historical)
- **All other stacks**: `compose.yml` with symlink `docker-compose.yml → compose.yml`

## Stack Descriptions

### Management Stack (`management/`)
Infrastructure services for monitoring and container management.

**Services**:
- **Portainer** (9000, 9443): Container management UI
- **Watchtower**: Automatic container updates (label-based)
- **Tautulli** (8181): Plex monitoring and statistics

**Network**: Bridge network `management`

### Media Stack (`media/`)
Media streaming services with Intel QuickSync hardware transcoding.

**Services**:
- **Plex** (32400): Media server, host network mode
- **Jellyfin** (8096): Alternative media server

**Hardware Access**: `/dev/dri:/dev/dri` for QuickSync
**Environment**: `LIBVA_DRIVER_NAME=iHD` for Jellyfin
**Storage**: NFS-mounted media at `/mnt/nas-media` (read-only)
**Network**: Plex uses host mode, Jellyfin on bridge network `media`

### Web Stack (`web/`)
User-facing web applications proxied through VPS.

**Services**:
- **NginxProxyManager** (80, 443, 81): Reverse proxy and SSL
- **Overseerr** (5055): Media request management
- **Wizarr** (5690): User invitation system
- **Homepage** (3000): Dashboard with Docker socket access

**Network**: Bridge network `web`
**Public Access**: Via WireGuard tunnel to VPS

### Cloud Services (`cloud/`)

#### Nextcloud (`cloud/nextcloud/`)
**Architecture**: Nextcloud AIO (All-in-One)
- Single mastercontainer manages all sub-containers
- Auto-deploys: Nextcloud, PostgreSQL, Redis, Apache, Collabora, Talk, etc.
- **Not** a multi-container compose stack

**Ports**:
- 8080: AIO admin interface
- 11000: Apache (configured via `APACHE_PORT`)

**Management**: Via AIO web interface at port 8080

#### Immich Main Services (`cloud/immich/`)
**Host**: NAB9 Mini PC

**Services**:
- **immich-server** (2283): Web UI and API
- **immich-microservices**: Background processing
- **PostgreSQL**: tensorchord/pgvecto-rs (vector extensions)
- **Redis**: Caching and queues

**Critical Configuration**:
```yaml
environment:
  - IMMICH_MACHINE_LEARNING_URL=http://UNRAID_IP:3003
```

**Network**: Bridge network `immich`

#### Immich-ML (`cloud/immich-ml/`)
**Host**: Unraid file server (NVIDIA P2000 GPU)

**Important**:
- This directory contains **setup documentation only**
- No compose stack runs on mini PC
- Actual service deployed via Docker web UI on Unraid
- Port 3003 must be accessible from mini PC

## Environment Variables

### Common Variables
```bash
PUID=1000                        # User ID for file ownership
PGID=1000                        # Group ID for file ownership
TZ=America/New_York              # Timezone
APPDATA_PATH=/var/lib/containers/appdata  # Application data root
```

### Stack-Specific Variables

**Media**:
```bash
PLEX_CLAIM_TOKEN=claim-xxxxx     # Plex server claim token
JELLYFIN_PUBLIC_URL=https://...  # Public Jellyfin URL
```

**Immich**:
```bash
UPLOAD_LOCATION=/var/lib/containers/appdata/immich/upload
DB_DATA_LOCATION=/var/lib/containers/appdata/immich/postgres
DB_PASSWORD=xxxxx
IMMICH_MACHINE_LEARNING_URL=http://192.168.1.x:3003
```

**Watchtower**:
```bash
WATCHTOWER_SCHEDULE=0 0 4 * * *  # Cron: daily at 4 AM
WATCHTOWER_NOTIFICATION_URL=...  # Shoutrrr notification URL
```

## Systemd Integration

### Service File Pattern
```ini
[Unit]
Description=Docker Compose [Stack] Stack
Requires=docker.service network-online.target
After=docker.service network-online.target
RequiresMountsFor=/mnt/nas-media  # If NFS required

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/srv/containers/[stack]
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=600

[Install]
WantedBy=multi-user.target
```

### Service Management
```bash
# Enable on boot
systemctl enable docker-compose-[stack].service

# Start/stop
systemctl start docker-compose-[stack].service
systemctl stop docker-compose-[stack].service

# Status and logs
systemctl status docker-compose-[stack].service
journalctl -u docker-compose-[stack].service -f
```

### Dependency Order
1. `docker-compose-management.service` (Portainer for monitoring)
2. `docker-compose-media.service`
3. `docker-compose-web.service`
4. `docker-compose-nextcloud.service`
5. `docker-compose-immich.service`

## Storage Architecture

### Local Storage
- **Path**: `/var/lib/containers/appdata`
- **Purpose**: Configuration, databases, caches
- **Ownership**: PUID/PGID (typically 1000:1000)
- **Persistence**: Survives container recreation

### NFS Mounts
- **Media Library**: `/mnt/nas-media` (read-only)
- **Source**: Unraid ZFS array
- **Usage**: Plex and Jellyfin media content
- **Systemd**: Must be mounted before starting media stack

### Environment Variable Pattern
```yaml
volumes:
  - ${APPDATA_PATH}/service:/config:z
  - /mnt/nas-media:/media:ro
```

## Hardware Transcoding

### Intel QuickSync Configuration

**Device Access**:
```yaml
devices:
  - /dev/dri:/dev/dri
```

**Jellyfin Environment**:
```yaml
environment:
  - LIBVA_DRIVER_NAME=iHD  # Intel iHD driver for 12th gen
```

**Benefits**:
- Hardware-accelerated H.264/H.265 encoding/decoding
- Low power consumption vs. GPU
- Multiple simultaneous transcodes

**Verification**:
```bash
ls -la /dev/dri/
# Should show: card0, renderD128
```

## Common Compose Patterns

### Standard Service Template
```yaml
services:
  servicename:
    image: namespace/image:tag
    container_name: servicename
    restart: unless-stopped
    environment:
      - PUID=${PUID:-1000}
      - PGID=${PGID:-1000}
      - TZ=${TZ:-America/New_York}
    ports:
      - "HOST:CONTAINER"
    volumes:
      - ${APPDATA_PATH}/servicename:/config:z
    networks:
      - stack_network
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
```

### Health Checks
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:PORT/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

### Dependency Management
```yaml
depends_on:
  - database
  - redis
```

## Watchtower Auto-Updates

### Enabled by Default
```yaml
labels:
  - "com.centurylinklabs.watchtower.enable=true"
```

### Disable for Specific Service
```yaml
labels:
  - "com.centurylinklabs.watchtower.enable=false"
```

### Update Schedule
- Default: Daily at 4 AM
- Configurable via `WATCHTOWER_SCHEDULE` in management stack
- Only updates containers with `watchtower.enable=true` label

## Integration Points

### WireGuard Tunnel (VPS ↔ Mini PC)
- Purpose: Secure public access to services
- Services exposed: Overseerr, Wizarr, Plex, Jellyfin
- Reverse proxy: NginxProxyManager on VPS

### Tailscale (Internal Network)
- Purpose: Administrative access
- Access: Portainer, Homepage, Tautulli
- No public exposure

### NFS (Unraid → Mini PC)
- Mount: `/mnt/nas-media`
- Content: Movies, TV shows, music
- Access: Read-only for Plex/Jellyfin
- Systemd: `RequiresMountsFor=/mnt/nas-media`

### Immich-ML Network Connection
- Mini PC Immich → Unraid Immich-ML (port 3003)
- Protocol: HTTP
- Configuration: `IMMICH_MACHINE_LEARNING_URL` environment variable
- Requirement: Low-latency LAN connection

## Migration Context

This deployment represents a **Plex migration from Unraid to CoreOS**:
- Original: Plex on Unraid file server
- Current: Parallel deployment on NAB9 mini PC
- Status: Testing phase with both systems operational
- Goal: Validate performance and stability before full cutover

**Implications**:
- Media library shared between both systems (NFS)
- Plex databases are independent
- Can switch back to Unraid if issues arise

## Troubleshooting Quick Reference

### Container Won't Start
```bash
# Check logs
docker logs <container_name>

# Verify compose syntax
cd /srv/containers/<stack>
docker compose config

# Check systemd status
systemctl status docker-compose-<stack>.service
```

### Permission Issues
```bash
# Verify ownership
ls -la ${APPDATA_PATH}/<service>

# Fix ownership
chown -R 1000:1000 ${APPDATA_PATH}/<service>
```

### NFS Mount Issues
```bash
# Verify mount
mount | grep nas-media

# Test NFS server
showmount -e <nfs_server_ip>

# Remount
umount /mnt/nas-media
mount /mnt/nas-media
```

### QuickSync Not Working
```bash
# Check device access
ls -la /dev/dri/

# Verify inside container
docker exec <container> ls -la /dev/dri/

# Check driver
docker exec jellyfin vainfo
```

### Immich-ML Connectivity
```bash
# Test from mini PC
curl http://<unraid_ip>:3003/ping

# Check Immich logs
docker logs immich_server | grep -i "machine learning"

# Verify ML service on Unraid
ssh unraid
docker ps | grep immich
```

## Key Operational Considerations

1. **Immutable OS**: CoreOS uses rpm-ostree. Package installation requires layering and reboot.
2. **Remote Access**: Physical access requires Marcus. Avoid risky changes.
3. **Systemd Management**: Services managed via systemd, not manual `docker compose` commands.
4. **NFS Dependencies**: Media stack requires NFS mount. Check systemd dependencies.
5. **Watchtower Updates**: Automatic updates enabled. Monitor for issues after 4 AM.
6. **Docker Runtime**: System uses Docker (not Podman) despite CoreOS typically using Podman.
7. **AIO Management**: Nextcloud sub-containers managed by AIO, not compose.
8. **Split Architecture**: Immich-ML runs on separate hardware. Network connectivity critical.

## Additional Resources

- **DEPLOYMENT.md**: Detailed deployment procedures and systemd setup
- **README.md**: Quick start and overview
- **agents.md**: Operational guidelines for AI coding assistants
- **Stack READMEs**: Service-specific documentation in each directory
- **UBlue CoreOS Docs**: https://universal-blue.org/
- **Immich Docs**: https://immich.app/docs
- **Nextcloud AIO Docs**: https://github.com/nextcloud/all-in-one
