# Claude AI Agent Guidelines for Docker Compose Stack

This document provides guidelines for Claude AI when working with this Docker Compose homelab stack.

## Important: SELinux Volume Mount Requirements

**CRITICAL**: This stack runs on **Fedora** with SELinux enabled. All volume mounts MUST include the `:z` tag for proper SELinux context.

### SELinux Volume Tag Rules

When creating or modifying Docker Compose files:

1. **Always add `:z` to bind-mounted volumes**
   ```yaml
   volumes:
     - ${APPDATA_PATH}/service:/config:z        # Correct
     - /mnt/data:/data:z                        # Correct
     - ${APPDATA_PATH}/service:/config          # WRONG - Missing :z
   ```

2. **Keep `:ro` for read-only mounts without `:z`**
   ```yaml
   volumes:
     - /var/run/docker.sock:/var/run/docker.sock:ro  # Correct - system socket
     - /etc/localtime:/etc/localtime:ro               # Correct - system file
   ```

3. **Named volumes do NOT need `:z`**
   ```yaml
   volumes:
     - model-cache:/cache      # Correct - named volume

   volumes:
     model-cache:
       name: immich_model_cache
   ```

### When to Use SELinux Tags

- ✅ **Use `:z`**: Application data directories (`${APPDATA_PATH}/*`)
- ✅ **Use `:z`**: User-writable mount points
- ✅ **Use `:z`**: Database data directories
- ✅ **Use `:z`**: Upload/storage directories
- ❌ **Do NOT use `:z`**: Docker socket (`/var/run/docker.sock`)
- ❌ **Do NOT use `:z`**: System files (read-only)
- ❌ **Do NOT use `:z`**: Named Docker volumes

### Examples from This Stack

**Web Stack** (nginxproxymanager):
```yaml
volumes:
  - ${APPDATA_PATH}/nginxproxymanager:/config:z  # Writable config
```

**Immich Stack**:
```yaml
volumes:
  - ${UPLOAD_LOCATION}:/usr/src/app/upload:z              # Uploads
  - ${DB_DATA_LOCATION}:/var/lib/postgresql/data:z        # Database
  - /etc/localtime:/etc/localtime:ro                       # System time (read-only)
```

**Management Stack** (portainer):
```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:ro   # Docker socket (read-only)
  - ${APPDATA_PATH}/portainer:/data:z               # Application data
```

## Repository Structure

```
/srv/containers/
├── media/          # Media streaming (Plex, Jellyfin)
├── web/            # Web applications (Nginx Proxy Manager, Overseerr)
├── management/     # Infrastructure (Portainer, Watchtower)
└── cloud/          # Cloud services
    ├── immich/       # Photo management (main services)
    ├── immich-ml/    # ML services with GPU
    └── nextcloud/    # File sync and share
```

## Stack Organization

### Media Stack
- **Location**: `media/`
- **Services**: Plex, Jellyfin
- **Hardware**: Intel QuickSync transcoding
- **Network**: Host network mode for Plex
- **Special Notes**: NFS-mounted media library at `/mnt/nas-media`

### Web Stack
- **Location**: `web/`
- **Services**: Nginx Proxy Manager, Overseerr, Wizarr, Homepage
- **Network**: Bridge network named `web`
- **Purpose**: User-facing web applications

### Management Stack
- **Location**: `management/`
- **Services**: Portainer, Watchtower, Tautulli
- **Network**: Bridge network named `management`
- **Purpose**: Infrastructure monitoring and container management

### Cloud Services (Split Deployment)

#### Immich Main Stack
- **Location**: `cloud/immich/`
- **Host**: Mini PC
- **Services**: Immich server, microservices, PostgreSQL, Redis
- **Database**: pgvecto-rs (PostgreSQL with vector extensions)
- **Important**: Set `IMMICH_MACHINE_LEARNING_URL` to point to fileserver ML service

#### Immich ML Stack
- **Location**: `cloud/immich-ml/`
- **Host**: Fileserver with NVIDIA P2000 GPU
- **Services**: Machine learning container only
- **Requirements**: NVIDIA Container Toolkit, CUDA support
- **Deployment**: Via Docker web UI (see `DOCKER-UI-DEPLOYMENT.md`)
- **Port**: 3003 (must be accessible from mini PC)

#### Nextcloud AIO
- **Location**: `cloud/nextcloud/`
- **Services**: All-in-One mastercontainer (auto-manages sub-containers)
- **Ports**: 8080 (admin), 11000 (Apache)
- **Special**: Uses AIO system that creates its own containers

## Environment Variables

Common variables used across all stacks:
- `PUID` - User ID (default: 1000)
- `PGID` - Group ID (default: 1000)
- `TZ` - Timezone (e.g., America/New_York)
- `APPDATA_PATH` - Base path for application data (typically `/var/lib/containers/appdata`)

## Development Guidelines

### When Adding New Services

1. **Check SELinux**: Add `:z` to all writable volume mounts
2. **Use existing patterns**: Follow the structure of similar services
3. **Add Watchtower label**: Include `com.centurylinklabs.watchtower.enable=true`
4. **Document**: Add README.md to stack directory if creating new stack
5. **Environment**: Create `.env.example` with documented variables
6. **Test**: Verify on Fedora with SELinux enforcing

### When Modifying Services

1. **Preserve SELinux tags**: Never remove `:z` from existing mounts
2. **Maintain network structure**: Keep services in their designated networks
3. **Check dependencies**: Consider service startup order
4. **Review systemd**: Update systemd service files if needed
5. **Update docs**: Keep README and DEPLOYMENT docs in sync

### Common Pitfalls to Avoid

❌ **Removing `:z` tags** - Will cause permission denied errors on Fedora
❌ **Adding `:z` to Docker socket** - Not needed for socket mounts
❌ **Adding `:z` to named volumes** - Not needed for Docker-managed volumes
❌ **Hardcoded paths** - Use environment variables instead
❌ **Missing .env.example** - Always provide example configuration
❌ **Undocumented variables** - Comment all env vars with purpose and examples

## Testing Changes

When making changes to compose files:

```bash
# Validate compose file syntax
docker compose config

# Check for SELinux denials (on Fedora)
sudo ausearch -m avc -ts recent

# Test service startup
docker compose up -d
docker compose ps
docker compose logs -f [service-name]

# Verify file permissions
ls -laZ ${APPDATA_PATH}/[service-name]
```

## Additional Resources

- [DEPLOYMENT.md](DEPLOYMENT.md) - Detailed deployment procedures
- [README.md](README.md) - Overview and quick start
- Stack-specific READMEs in each directory
- [Docker SELinux Documentation](https://docs.docker.com/storage/bind-mounts/#configure-the-selinux-label)
