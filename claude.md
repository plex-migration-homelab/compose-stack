# Claude AI Guidelines for Docker Compose Homelab Stack

This document provides guidelines for Claude AI when working with this Docker Compose homelab stack.

## ⚠️ CRITICAL: SELinux Volume Mount Requirements

**MANDATORY**: This stack runs on **Fedora CoreOS** with **SELinux enabled in enforcing mode**.

### All Volume Mounts MUST Follow SELinux Rules

#### Rule 1: Bind Mounts Need `:z` Tag

When mounting host directories into containers, **always** add the `:z` SELinux context flag:

```yaml
# ✅ CORRECT
volumes:
  - ${APPDATA_PATH}/service:/config:z
  - /mnt/storage:/data:z
  - ${UPLOAD_LOCATION}:/uploads:z

# ❌ INCORRECT - Will fail with "permission denied" on Fedora CoreOS
volumes:
  - ${APPDATA_PATH}/service:/config
  - /mnt/storage:/data
```

**Why?** The `:z` flag tells Docker to relabel the host directory with the appropriate SELinux context (`svirt_sandbox_file_t`), allowing containers to read/write files.

#### Rule 2: Read-Only System Mounts Use `:ro` (Without `:z`)

System files and the Docker socket should use `:ro` (read-only) without `:z`:

```yaml
# ✅ CORRECT
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:ro
  - /etc/localtime:/etc/localtime:ro
  - /mnt/nas-media:/media:ro  # Read-only NFS mount
```

These system files already have correct SELinux labels and should not be relabeled.

#### Rule 3: Named Volumes Don't Need Tags

Docker-managed named volumes don't need SELinux tags:

```yaml
# ✅ CORRECT
volumes:
  - postgres_data:/var/lib/postgresql/data
  - model-cache:/cache

volumes:
  postgres_data:
    name: service_postgres_data
  model-cache:
    name: immich_model_cache
```

Named volumes are automatically labeled correctly by Docker.

### SELinux Tag Quick Reference

| Mount Type | SELinux Tag | Example |
|------------|-------------|---------|
| Application data directory | `:z` | `${APPDATA_PATH}/plex:/config:z` |
| Database data directory | `:z` | `${DB_DATA_LOCATION}:/var/lib/postgresql/data:z` |
| Upload/storage directory | `:z` | `${UPLOAD_PATH}:/uploads:z` |
| Transcode/cache directory | `:z` | `${APPDATA_PATH}/plex/transcode:/transcode:z` |
| Docker socket | `:ro` only | `/var/run/docker.sock:/var/run/docker.sock:ro` |
| System time files | `:ro` only | `/etc/localtime:/etc/localtime:ro` |
| Read-only NFS media | `:ro` only | `/mnt/nas-media:/media:ro` |
| Named Docker volumes | None | `volume_name:/path` |

## System Architecture

### Hardware Environment
- **NAB9 Mini PC** (Primary host)
  - CPU: 12th gen Intel i5 with QuickSync
  - RAM: 32GB
  - OS: **Fedora CoreOS** (immutable, rpm-ostree based, **SELinux enforcing**)
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
└── agents.md                    # Same content as this file
```

### File Naming Convention
- **Management stack**: Uses single `docker-compose.yml` (historical)
- **All other stacks**: `compose.yml` with symlink `docker-compose.yml → compose.yml`

## Stack Descriptions with SELinux Examples

### Management Stack (`management/docker-compose.yml`)

Infrastructure services for monitoring and container management.

**Services**:
- **Portainer** (9000, 9443): Container management UI
- **Watchtower**: Automatic container updates (label-based, runs at 4 AM)
- **Tautulli** (8181): Plex monitoring and statistics

**SELinux Volume Patterns**:
```yaml
# Portainer
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:ro  # Docker socket (no :z)
  - ${APPDATA_PATH}/portainer:/data:z              # Application data (needs :z)

# Watchtower
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:ro  # Docker socket only

# Tautulli
volumes:
  - ${APPDATA_PATH}/tautulli:/config:z             # Configuration (needs :z)
```

### Media Stack (`media/compose.yml`)

Media streaming services with Intel QuickSync hardware transcoding.

**Services**:
- **Plex** (32400): Media server, host network mode
- **Jellyfin** (8096): Alternative media server

**SELinux Volume Patterns**:
```yaml
# Plex
volumes:
  - ${APPDATA_PATH}/plex:/config:z                # Configuration (needs :z)
  - /mnt/nas-media:/media:ro                      # NFS read-only (no :z)
  - ${APPDATA_PATH}/plex/transcode:/transcode:z   # Transcode cache (needs :z)

# Jellyfin
volumes:
  - ${APPDATA_PATH}/jellyfin/config:/config:z     # Configuration (needs :z)
  - ${APPDATA_PATH}/jellyfin/cache:/cache:z       # Cache (needs :z)
  - ${APPDATA_PATH}/jellyfin/transcode:/transcode:z  # Transcode (needs :z)
  - /mnt/nas-media:/media:ro                      # NFS read-only (no :z)
```

**Hardware Access**: `/dev/dri:/dev/dri` for QuickSync
**Environment**: `LIBVA_DRIVER_NAME=iHD` for Jellyfin (12th gen Intel)
**Network**: Plex uses host mode, Jellyfin on bridge network `media`

### Web Stack (`web/compose.yml`)

User-facing web applications proxied through VPS.

**Services**:
- **NginxProxyManager** (80, 443, 81): Reverse proxy and SSL
- **Overseerr** (5055): Media request management
- **Wizarr** (5690): User invitation system
- **Homepage** (3000): Dashboard with Docker socket access

**SELinux Volume Patterns**:
```yaml
# NginxProxyManager
volumes:
  - ${APPDATA_PATH}/nginxproxymanager:/config:z   # Configuration (needs :z)

# Overseerr
volumes:
  - ${APPDATA_PATH}/overseerr:/config:z           # Configuration (needs :z)

# Wizarr
volumes:
  - ${APPDATA_PATH}/wizarr:/data/database:z       # Database (needs :z)

# Homepage
volumes:
  - ${APPDATA_PATH}/homepage:/app/config:z        # Configuration (needs :z)
  - /var/run/docker.sock:/var/run/docker.sock:ro  # Docker socket (no :z)
```

**Network**: Bridge network `web`
**Public Access**: Via WireGuard tunnel to VPS

### Cloud Services: Nextcloud AIO (`cloud/nextcloud/compose.yml`)

**Architecture**: Nextcloud All-in-One (single mastercontainer)
- **NOT** a multi-container compose stack
- Mastercontainer auto-creates sub-containers via Docker API
- Sub-containers: `nextcloud-aio-*` (PostgreSQL, Redis, Apache, Collabora, Talk, etc.)
- Management via AIO web UI at port 8080

**SELinux Volume Pattern**:
```yaml
services:
  nextcloud-aio-mastercontainer:
    image: nextcloud/all-in-one:latest
    init: true  # Required for AIO
    ports:
      - "8080:8080"  # AIO admin interface
    environment:
      - APACHE_PORT=11000
      - APACHE_IP_BINDING=0.0.0.0
    volumes:
      - nextcloud_aio_mastercontainer:/mnt/docker-aio-config  # Named volume (no tags)
      - /var/run/docker.sock:/var/run/docker.sock:ro          # Docker socket (no :z)

volumes:
  nextcloud_aio_mastercontainer:
    name: nextcloud_aio_mastercontainer
```

**Critical**:
- Don't add PostgreSQL/Redis containers (AIO handles this)
- Don't modify sub-containers directly (use AIO interface)
- Named volume for mastercontainer (no `:z` needed)

### Cloud Services: Immich Split Deployment

**Files**:
- `cloud/immich/compose.yml` - Main services (mini PC)
- `cloud/immich-ml/` - Documentation only (service runs on Unraid)

**Architecture**: Distributed deployment
- **Mini PC**: Web UI, API, microservices, PostgreSQL, Redis
- **Unraid**: ML container with NVIDIA P2000 GPU

**SELinux Volume Patterns**:
```yaml
# Immich Server & Microservices
volumes:
  - ${UPLOAD_LOCATION}:/usr/src/app/upload:z   # Photo uploads (needs :z)
  - /etc/localtime:/etc/localtime:ro            # System time (no :z)

# PostgreSQL (pgvecto-rs with vector extensions)
volumes:
  - ${DB_DATA_LOCATION}:/var/lib/postgresql/data:z  # Database data (needs :z)

# Redis (cache only, no persistence needed)
# No volumes required
```

**Critical Configuration**:
```yaml
environment:
  - IMMICH_MACHINE_LEARNING_URL=http://UNRAID_IP:3003  # Points to Unraid ML service
```

**Important**:
- Never add ML container to mini PC compose (runs on Unraid)
- Use `tensorchord/pgvecto-rs` image, not standard PostgreSQL
- Both server and microservices need `IMMICH_MACHINE_LEARNING_URL`
- Firewall on Unraid must allow mini PC access to port 3003

## Common Compose Patterns with SELinux

### Standard Service Template
```yaml
services:
  servicename:
    image: namespace/image:latest
    container_name: servicename
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true  # For internet-facing services
    environment:
      - PUID=${PUID:-1000}
      - PGID=${PGID:-1000}
      - TZ=${TZ:-America/New_York}
    ports:
      - "HOST:CONTAINER"
    volumes:
      - ${APPDATA_PATH}/servicename:/config:z  # Writable bind mount needs :z
      - /etc/localtime:/etc/localtime:ro       # Read-only system file
    networks:
      - stack_network
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
```

### Database Service Pattern
```yaml
services:
  database:
    image: postgres:16-alpine
    container_name: service_postgres
    restart: unless-stopped
    environment:
      - POSTGRES_USER=${DB_USERNAME:-postgres}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=${DB_DATABASE_NAME:-servicedb}
    volumes:
      - ${DB_DATA_LOCATION}:/var/lib/postgresql/data:z  # Database needs :z
    networks:
      - service_network
    # No external ports - internal access only
```

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
IMMICH_MACHINE_LEARNING_URL=http://192.168.1.x:3003  # Unraid ML service
```

**Watchtower**:
```bash
WATCHTOWER_SCHEDULE=0 0 4 * * *  # Cron: daily at 4 AM
WATCHTOWER_NOTIFICATION_URL=...  # Shoutrrr notification URL
```

## Systemd Integration

Services are managed via systemd, not manual `docker compose` commands.

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

## Common SELinux Issues and Fixes

### Issue 1: Permission Denied Errors

**Symptom:**
```
Error: EACCES: permission denied, open '/config/config.yaml'
mkdir: cannot create directory '/data': Permission denied
```

**Cause:** Missing `:z` tag on volume mount

**Fix:**
```yaml
# Before (broken on Fedora CoreOS)
volumes:
  - ${APPDATA_PATH}/service:/config

# After (fixed)
volumes:
  - ${APPDATA_PATH}/service:/config:z
```

### Issue 2: SELinux Denial Logs

**Check for SELinux denials:**
```bash
# View recent SELinux denials
sudo ausearch -m avc -ts recent

# Check SELinux context of directory
ls -Z /var/lib/containers/appdata/service

# Should see: system_u:object_r:svirt_sandbox_file_t:s0
# If you see: unconfined_u:object_r:var_lib_t:s0 - needs :z tag
```

**Manually relabel if needed** (usually not necessary):
```bash
sudo chcon -Rt svirt_sandbox_file_t /var/lib/containers/appdata/service
```

### Issue 3: Docker Socket Access Denied

**Symptom:**
```
Error response from daemon: dial unix /var/run/docker.sock: permission denied
```

**Cause:** Incorrectly added `:z` to Docker socket mount

**Fix:**
```yaml
# Wrong (causes issues)
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:z

# Correct
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:ro
```

## Troubleshooting Quick Reference

### Container Won't Start
```bash
# Check logs
docker logs <container_name>

# Verify compose syntax
cd /srv/containers/<stack>
docker compose config

# Check for SELinux denials
sudo ausearch -m avc -ts recent | grep denied

# Verify file permissions and SELinux context
ls -laZ ${APPDATA_PATH}/<service>
```

### Permission Issues
```bash
# Check ownership
ls -la ${APPDATA_PATH}/<service>

# Fix ownership (if needed)
chown -R 1000:1000 ${APPDATA_PATH}/<service>

# Check SELinux context
ls -Z ${APPDATA_PATH}/<service>
```

### QuickSync Not Working
```bash
# Check device access
ls -la /dev/dri/

# Verify inside container
docker exec <container> ls -la /dev/dri/

# Check driver (Jellyfin)
docker exec jellyfin vainfo
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

## Critical Warnings

### ⚠️ NEVER Do These

1. **Remove `:z` tags from writable volume mounts**
   - Will cause "permission denied" errors on Fedora CoreOS
   - Containers won't be able to write to mounted directories

2. **Add `:z` to Docker socket mounts**
   - Will cause permission issues
   - Always use `:ro` only for `/var/run/docker.sock`

3. **Add `:z` to named Docker volumes**
   - Not needed for Docker-managed volumes
   - Docker handles SELinux labels automatically

4. **Change Plex to bridge network**
   - Breaks device discovery
   - Breaks remote access
   - Keep host mode

5. **Add ML container to mini PC Immich stack**
   - Mini PC has no GPU
   - Breaks documented architecture
   - CPU-only ML is slow

6. **Modify Nextcloud sub-containers directly**
   - AIO will override changes
   - May break AIO management
   - Use AIO interface instead

7. **Modify systemd units without testing**
   - Remote system can't be recovered easily
   - Test in duplicate environment first

8. **Change network configuration**
   - WireGuard tunnel disruption = service outage
   - Always have out-of-band access plan

## Key Operational Considerations

1. **SELinux Enforcing**: All writable bind mounts MUST have `:z` tag
2. **Immutable OS**: Fedora CoreOS uses rpm-ostree. Package installation requires layering and reboot.
3. **Remote Access**: Physical access requires Marcus. Avoid risky changes.
4. **Systemd Management**: Services managed via systemd, not manual `docker compose` commands.
5. **NFS Dependencies**: Media stack requires NFS mount. Check systemd dependencies.
6. **Watchtower Updates**: Automatic updates enabled. Monitor for issues after 4 AM.
7. **Docker Runtime**: System uses Docker (not Podman).
8. **AIO Management**: Nextcloud sub-containers managed by AIO, not compose.
9. **Split Architecture**: Immich-ML runs on separate hardware. Network connectivity critical.

## Validation Steps Before Applying Changes

**Always validate**:
```bash
# 1. Compose syntax
docker compose config

# 2. Environment variables resolved
docker compose config | grep -A 5 environment

# 3. Volume paths exist
ls -la ${APPDATA_PATH}/<service>

# 4. SELinux contexts correct
ls -Z ${APPDATA_PATH}/<service>

# 5. No port conflicts
netstat -tlnp | grep <port>

# 6. Check for SELinux denials after start
docker compose up -d
sudo ausearch -m avc -ts recent
```

## Additional Resources

- **DEPLOYMENT.md**: Detailed deployment procedures and systemd setup
- **README.md**: Quick start and overview
- **agents.md**: Same content as this file
- **Stack READMEs**: Service-specific documentation in each directory
- **Fedora CoreOS Docs**: https://docs.fedoraproject.org/en-US/fedora-coreos/
- **Docker SELinux Docs**: https://docs.docker.com/storage/bind-mounts/#configure-the-selinux-label
- **Immich Docs**: https://immich.app/docs
- **Nextcloud AIO Docs**: https://github.com/nextcloud/all-in-one
