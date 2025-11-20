# AI Agent Guidelines for Docker Compose Homelab

This document provides guidelines for AI coding assistants (Claude, Copilot, etc.) when working with this Docker Compose stack.

## Critical: Fedora SELinux Requirements

**⚠️ MANDATORY**: This stack runs on **Fedora Linux** with **SELinux enabled in enforcing mode**.

### SELinux Volume Mount Rules

All container volume mounts MUST follow these SELinux labeling rules:

#### Rule 1: Bind Mounts Need `:z` Tag

When mounting host directories into containers, **always** add the `:z` SELinux context flag:

```yaml
# ✅ CORRECT
volumes:
  - ${APPDATA_PATH}/service:/config:z
  - /mnt/storage:/data:z
  - /opt/app/data:/app/data:z

# ❌ INCORRECT - Will fail with permission denied on Fedora
volumes:
  - ${APPDATA_PATH}/service:/config
  - /mnt/storage:/data
```

**Why?** The `:z` flag tells Docker to relabel the host directory with the appropriate SELinux context (`svirt_sandbox_file_t`), allowing containers to read/write the files.

#### Rule 2: Read-Only System Mounts Keep `:ro`

System files and the Docker socket should use `:ro` (read-only) without `:z`:

```yaml
# ✅ CORRECT
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:ro
  - /etc/localtime:/etc/localtime:ro
  - /etc/timezone:/etc/timezone:ro
```

These system files already have correct SELinux labels and should not be relabeled.

#### Rule 3: Named Volumes Don't Need Tags

Docker-managed named volumes don't need SELinux tags:

```yaml
# ✅ CORRECT
volumes:
  - postgres_data:/var/lib/postgresql/data
  - redis_cache:/data

volumes:
  postgres_data:
    name: service_postgres_data
  redis_cache:
    name: service_redis_cache
```

Named volumes are automatically labeled correctly by Docker.

### SELinux Tag Quick Reference

| Mount Type | SELinux Tag | Example |
|------------|-------------|---------|
| Application data directory | `:z` | `${APPDATA_PATH}/plex:/config:z` |
| Database data directory | `:z` | `/var/lib/appdata/db:/var/lib/postgresql/data:z` |
| Upload/storage directory | `:z` | `${UPLOAD_PATH}:/uploads:z` |
| Cache directory | `:z` | `/opt/cache:/cache:z` |
| Docker socket | `:ro` only | `/var/run/docker.sock:/var/run/docker.sock:ro` |
| System time | `:ro` only | `/etc/localtime:/etc/localtime:ro` |
| Read-only media (NFS) | `:ro` only | `/mnt/nas-media:/media:ro` |
| Named volumes | None | `volume_name:/path` |

## Stack-Specific Considerations

### Media Stack (Plex, Jellyfin)

```yaml
# Already has correct SELinux tags
volumes:
  - ${APPDATA_PATH}/plex:/config:z              # Application data
  - /mnt/nas-media:/media:ro                    # Read-only NFS mount
  - ${APPDATA_PATH}/plex/transcode:/transcode:z # Transcode cache
```

**Notes:**
- Plex uses host network mode
- Both services require `/dev/dri` device for hardware transcoding
- NFS mounts are read-only and don't need `:z`

### Web Stack (Nginx Proxy Manager, Overseerr, Wizarr, Homepage)

```yaml
# Pattern to follow
volumes:
  - ${APPDATA_PATH}/[service]:/config:z            # Configuration
  - /var/run/docker.sock:/var/run/docker.sock:ro  # Docker access (homepage)
```

**Notes:**
- All services in bridge network named `web`
- Homepage needs Docker socket access for monitoring

### Management Stack (Portainer, Watchtower, Tautulli)

```yaml
# Portainer pattern
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:ro  # Docker API access
  - ${APPDATA_PATH}/portainer:/data:z              # Portainer data

# Watchtower pattern
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:ro  # Docker API access only

# Tautulli pattern
volumes:
  - ${APPDATA_PATH}/tautulli:/config:z             # Configuration
```

**Notes:**
- Portainer and Watchtower need Docker socket access
- All services in bridge network named `management`

### Cloud Stack (Immich, Nextcloud)

#### Immich Main Services (Mini PC)

```yaml
# Server and Microservices pattern
volumes:
  - ${UPLOAD_LOCATION}:/usr/src/app/upload:z   # Photo uploads
  - /etc/localtime:/etc/localtime:ro            # System time

# Database pattern
volumes:
  - ${DB_DATA_LOCATION}:/var/lib/postgresql/data:z  # PostgreSQL data
```

**Notes:**
- Split deployment: main services on mini PC, ML on fileserver
- Requires `IMMICH_MACHINE_LEARNING_URL` pointing to fileserver:3003
- Uses pgvecto-rs (PostgreSQL with vector extensions)

#### Immich ML (Fileserver with GPU)

```yaml
# ML container pattern
volumes:
  - model-cache:/cache  # Named volume - no tag needed
```

**Notes:**
- Deployed via Docker UI on fileserver with NVIDIA P2000
- Requires `--gpus all` and NVIDIA Container Toolkit
- Named volume for model caching

#### Nextcloud All-in-One

```yaml
# Mastercontainer pattern
volumes:
  - nextcloud_aio_mastercontainer:/mnt/docker-aio-config  # Named volume
  - /var/run/docker.sock:/var/run/docker.sock:ro          # Docker access
```

**Notes:**
- Uses AIO system that auto-creates sub-containers
- Sub-containers managed by mastercontainer, not compose

## Creating New Services

When adding a new service to any stack, follow this checklist:

### 1. Volume Mounts
- [ ] Add `:z` to all bind-mounted application data directories
- [ ] Keep `:ro` for read-only system files
- [ ] Use named volumes for large datasets if appropriate
- [ ] Verify no `:z` on Docker socket mounts

### 2. Standard Configuration
```yaml
services:
  servicename:
    image: [image:tag]
    container_name: servicename
    restart: unless-stopped
    environment:
      - PUID=${PUID:-1000}
      - PGID=${PGID:-1000}
      - TZ=${TZ:-America/New_York}
    ports:
      - "PORT:PORT"
    volumes:
      - ${APPDATA_PATH}/servicename:/config:z
    networks:
      - [stack_network]
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
```

### 3. Documentation
- [ ] Add service to stack's README.md
- [ ] Create `.env.example` with new variables
- [ ] Document ports used
- [ ] Note any special requirements (hardware, dependencies)

### 4. Testing on Fedora
```bash
# Validate compose syntax
docker compose config

# Check for SELinux denials
sudo ausearch -m avc -ts recent | grep denied

# Start service
docker compose up -d servicename

# Check logs for permission errors
docker compose logs -f servicename

# Verify file access
ls -laZ ${APPDATA_PATH}/servicename
```

## Common SELinux Issues and Fixes

### Issue 1: Permission Denied Errors

**Symptom:**
```
Error: EACCES: permission denied, open '/config/config.yaml'
Permission denied: /data
```

**Cause:** Missing `:z` tag on volume mount

**Fix:**
```yaml
# Before (broken)
volumes:
  - ${APPDATA_PATH}/service:/config

# After (fixed)
volumes:
  - ${APPDATA_PATH}/service:/config:z
```

### Issue 2: Container Can't Write to Mounted Directory

**Symptom:**
```
mkdir: cannot create directory '/data/cache': Permission denied
```

**Diagnosis:**
```bash
# Check SELinux context
ls -Z /var/lib/containers/appdata/service

# Should see: system_u:object_r:svirt_sandbox_file_t:s0
# If you see: unconfined_u:object_r:var_lib_t:s0 - needs :z tag
```

**Fix:** Add `:z` to the volume mount

### Issue 3: Docker Socket Access Denied

**Symptom:**
```
Error response from daemon: dial unix /var/run/docker.sock: connect: permission denied
```

**Cause:** Incorrectly added `:z` to Docker socket mount

**Fix:**
```yaml
# Wrong
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:z

# Correct
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:ro
```

**Alternative:** Add container user to docker group (not recommended for socket mounts)

## Environment Best Practices

### Standard Variables

Always support these environment variables:
```yaml
environment:
  - PUID=${PUID:-1000}      # User ID for file ownership
  - PGID=${PGID:-1000}      # Group ID for file ownership
  - TZ=${TZ:-America/New_York}  # Timezone
```

### Service-Specific Variables

Document in `.env.example`:
```bash
# Service Configuration
SERVICE_PORT=8080          # Web interface port
SERVICE_DATA=/path/to/data # Data directory

# Database Settings (if applicable)
DB_PASSWORD=changeme       # Database password
DB_USERNAME=appuser        # Database username
DB_DATABASE_NAME=appdb     # Database name

# Feature Flags
ENABLE_FEATURE=true        # Enable specific feature
```

## Network Architecture

### Bridge Networks (Default)

Each stack has its own bridge network:
- `media` - Media services
- `web` - Web-facing services
- `management` - Infrastructure services
- `immich` - Immich main services
- `immich_ml` - Immich ML services
- `nextcloud_aio` - Nextcloud services

### Host Network Mode

Only used for Plex (requires direct network access for discovery and remote access).

### Cross-Stack Communication

If services need to communicate across stacks, add them to multiple networks:
```yaml
networks:
  - web
  - management
```

## Automated Updates with Watchtower

All services include the Watchtower label:
```yaml
labels:
  - "com.centurylinklabs.watchtower.enable=true"
```

This enables automatic container updates. Exclude services from auto-update:
```yaml
labels:
  - "com.centurylinklabs.watchtower.enable=false"
```

## Additional Resources

- **Main Documentation**: [README.md](README.md)
- **Deployment Guide**: [DEPLOYMENT.md](DEPLOYMENT.md)
- **Claude AI Guide**: [claude.md](claude.md)
- **Stack READMEs**: See each stack directory
- **Docker SELinux Docs**: https://docs.docker.com/storage/bind-mounts/#configure-the-selinux-label
- **Fedora SELinux Guide**: https://docs.fedoraproject.org/en-US/quick-docs/getting-started-with-selinux/

## Quick Command Reference

```bash
# Validate compose file with SELinux context
docker compose config

# Check SELinux context of directory
ls -Z /var/lib/containers/appdata/service

# View SELinux denials
sudo ausearch -m avc -ts recent

# Manually relabel directory (if needed)
sudo chcon -Rt svirt_sandbox_file_t /var/lib/containers/appdata/service

# Restore default SELinux labels
sudo restorecon -Rv /var/lib/containers/appdata

# Start service with logs
docker compose up -d && docker compose logs -f

# Check file permissions inside container
docker exec servicename ls -la /config
```

## Version History

- **2025-11-20**: Added comprehensive SELinux `:z` tag requirements for Fedora compatibility
- Initial setup with media, web, management, and cloud stacks
