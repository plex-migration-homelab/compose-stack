# AI Agent Operational Guidelines: Docker Compose Homelab

Directive guidelines for AI coding assistants (Claude, Copilot, etc.) modifying this UBlue CoreOS Docker Compose stack.

## Critical Context Awareness

### ⚠️ Remote Hosting Constraint
**This system is physically remote.** Boot failures, systemd misconfigurations, or network issues require manual intervention by Marcus at the physical location.

**Implications**:
- **Stability over features**: Never sacrifice system reliability for convenience
- **Test before deploy**: Validate changes thoroughly before applying
- **Avoid risky operations**: No kernel modules, no bootloader changes, no network reconfigurations
- **Rollback plan**: Always know how to revert changes

### Immutable Operating System
**UBlue CoreOS** uses rpm-ostree (immutable filesystem).

**What this means**:
- Can't use `apt`/`dnf`/`yum` for traditional package installation
- Package changes require `rpm-ostree install` + reboot
- System files in `/usr` are read-only
- Container-first philosophy: **prefer containerized solutions**

**Decision Tree for Package Needs**:
```
Need a tool/service?
├─ Available as container? → Use Docker container ✅
├─ Available in rpm-ostree? → rpm-ostree install (requires reboot) ⚠️
└─ Only source/binary? → Reconsider approach or use toolbox ❌
```

### Docker (Not Podman) Runtime
This system runs **Docker**, not Podman (despite CoreOS typically using Podman).

**Implications**:
- Use `docker` commands, not `podman`
- Systemd service files reference `/usr/bin/docker`
- Socket path: `/var/run/docker.sock`
- Compose: `docker compose` (v2 syntax)

## Modification Principles

### 1. Preserve Architecture Patterns

**Follow existing patterns** in the stack:

```yaml
# ✅ CORRECT: Follow established pattern
services:
  newservice:
    image: namespace/image:latest
    container_name: newservice
    restart: unless-stopped
    environment:
      - PUID=${PUID:-1000}
      - PGID=${PGID:-1000}
      - TZ=${TZ:-America/New_York}
    volumes:
      - ${APPDATA_PATH}/newservice:/config:z
    networks:
      - web
    labels:
      - "com.centurylinklabs.watchtower.enable=true"

# ❌ INCORRECT: Inconsistent with stack patterns
services:
  newservice:
    image: namespace/image:latest
    restart: always  # Use unless-stopped
    volumes:
      - /opt/data:/config  # Use ${APPDATA_PATH}
    # Missing: PUID/PGID, TZ, network, labels
```

### 2. Maintain Security Posture

**Security Requirements**:
- ✅ Use `security_opt: [no-new-privileges:true]` for internet-facing services
- ✅ Mount Docker socket read-only: `/var/run/docker.sock:ro`
- ✅ Use environment variables for secrets (via `.env` files)
- ✅ Prefer `restart: unless-stopped` over `restart: always`
- ❌ Never hardcode passwords/tokens in compose files
- ❌ Never commit `.env` files to git
- ❌ Avoid `privileged: true` unless absolutely necessary

**Example**:
```yaml
# ✅ CORRECT: Secure configuration
services:
  nginxproxymanager:
    security_opt:
      - no-new-privileges:true
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      - DB_PASSWORD=${DB_PASSWORD}

# ❌ INCORRECT: Insecure patterns
services:
  nginxproxymanager:
    privileged: true  # Unnecessary privilege escalation
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock  # Writable socket
    environment:
      - DB_PASSWORD=supersecret123  # Hardcoded secret
```

### 3. Environment Variable Consistency

**Standard Variables (all services)**:
```yaml
environment:
  - PUID=${PUID:-1000}
  - PGID=${PGID:-1000}
  - TZ=${TZ:-America/New_York}
```

**Path Variables**:
```yaml
volumes:
  - ${APPDATA_PATH}/service:/config:z
  - /mnt/nas-media:/media:ro  # NFS paths are absolute
```

**Service-Specific Variables**:
```yaml
# Document in .env.example with comments
environment:
  - SERVICE_PORT=${SERVICE_PORT:-8080}  # Default port
  - DB_PASSWORD=${DB_PASSWORD}           # Required, no default
  - FEATURE_FLAG=${FEATURE_FLAG:-false}  # Optional feature
```

### 4. Volume Mount Best Practices

**SELinux Context (`:z` tag)**:
```yaml
# ✅ CORRECT: Writable bind mounts need :z
volumes:
  - ${APPDATA_PATH}/service:/config:z
  - ${APPDATA_PATH}/service/cache:/cache:z
  - ${UPLOAD_PATH}:/uploads:z

# ✅ CORRECT: Read-only system files don't need :z
volumes:
  - /etc/localtime:/etc/localtime:ro
  - /var/run/docker.sock:/var/run/docker.sock:ro
  - /mnt/nas-media:/media:ro  # Read-only NFS

# ✅ CORRECT: Named volumes don't need :z
volumes:
  - postgres_data:/var/lib/postgresql/data

# ❌ INCORRECT: Missing :z on writable bind mount
volumes:
  - ${APPDATA_PATH}/service:/config  # Will cause permission errors
```

**Volume Guidelines**:
- **Writable bind mounts**: Always use `:z`
- **Read-only mounts**: Use `:ro`, skip `:z`
- **Named volumes**: No tags needed
- **Docker socket**: Always `:ro`, never `:z`

## Stack-Specific Implementation Details

### Management Stack

**File**: `management/docker-compose.yml` (NOT `compose.yml`)

**Services**:
- **Portainer**: Docker management UI, needs socket access
- **Watchtower**: Auto-updates, label-based, runs at 4 AM
- **Tautulli**: Plex statistics, needs Plex network access

**Key Patterns**:
```yaml
# Portainer/Watchtower: Docker socket access
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:ro

# Watchtower: Label-based updates
environment:
  - WATCHTOWER_LABEL_ENABLE=true
  - WATCHTOWER_CLEANUP=true
  - WATCHTOWER_SCHEDULE=0 0 4 * * *  # Daily 4 AM

# All services: Enable auto-update
labels:
  - "com.centurylinklabs.watchtower.enable=true"
```

**Adding New Infrastructure Services**:
1. Add to `management` network
2. Include Watchtower label
3. If needs Docker access, mount socket read-only
4. Update systemd service if dependencies change

### Media Stack

**File**: `media/compose.yml` (symlink: `docker-compose.yml`)

**Architecture**:
- **Plex**: Host network mode (required for discovery/remote access)
- **Jellyfin**: Bridge network, explicit port mapping
- **Both**: QuickSync hardware transcoding via `/dev/dri`

**Critical Patterns**:
```yaml
# Plex: Host network mode
plex:
  network_mode: host
  # No ports section with host mode
  devices:
    - /dev/dri:/dev/dri

# Jellyfin: Bridge network + hardware transcode
jellyfin:
  ports:
    - "8096:8096"
  devices:
    - /dev/dri:/dev/dri
  environment:
    - LIBVA_DRIVER_NAME=iHD  # Required for 12th gen Intel

# Both: NFS media library
volumes:
  - /mnt/nas-media:/media:ro  # Read-only, no :z
```

**When Modifying Media Stack**:
- ⚠️ Never change Plex to bridge mode (breaks discovery)
- ⚠️ Never remove `/dev/dri` device (breaks transcoding)
- ⚠️ Keep `LIBVA_DRIVER_NAME=iHD` for Jellyfin
- ⚠️ NFS mount must be available before stack starts
- ✅ Systemd service needs `RequiresMountsFor=/mnt/nas-media`

### Web Stack

**File**: `web/compose.yml` (symlink: `docker-compose.yml`)

**Services**:
- **NginxProxyManager**: Reverse proxy, manages SSL certificates
- **Overseerr**: Media requests, connects to Plex/Jellyfin
- **Wizarr**: User invitations
- **Homepage**: Dashboard, needs Docker socket for monitoring

**Network Architecture**:
- All services on `web` bridge network
- Accessible via WireGuard tunnel from VPS
- VPS runs NginxProxyManager for public access

**Key Patterns**:
```yaml
# Homepage: Needs Docker socket for container status
homepage:
  volumes:
    - ${APPDATA_PATH}/homepage:/app/config:z
    - /var/run/docker.sock:/var/run/docker.sock:ro
  networks:
    - web

# Services accessible via reverse proxy
overseerr:
  networks:
    - web
  # Internal port, proxied through VPS
  ports:
    - "5055:5055"
```

**Cross-Stack Communication**:
If a web service needs to access media services:
```yaml
networks:
  - web
  - media  # Add second network for cross-stack access
```

### Cloud Services: Nextcloud AIO

**File**: `cloud/nextcloud/compose.yml`

**Architecture**: Nextcloud All-in-One (single mastercontainer)

**Critical Understanding**:
- **NOT** a multi-container compose stack
- Mastercontainer auto-creates sub-containers via Docker API
- Sub-containers: `nextcloud-aio-*` (PostgreSQL, Redis, Apache, Collabora, etc.)
- Management via AIO web UI at port 8080

**Compose Pattern**:
```yaml
services:
  nextcloud-aio-mastercontainer:
    image: nextcloud/all-in-one:latest
    init: true  # Required for AIO
    ports:
      - "8080:8080"  # AIO admin interface
    environment:
      - APACHE_PORT=11000  # Internal port for Nextcloud
      - APACHE_IP_BINDING=0.0.0.0
    volumes:
      - nextcloud_aio_mastercontainer:/mnt/docker-aio-config  # Named volume
      - /var/run/docker.sock:/var/run/docker.sock:ro  # Creates sub-containers
```

**When Modifying Nextcloud**:
- ❌ Don't add PostgreSQL/Redis containers (AIO handles this)
- ❌ Don't change the mastercontainer volume name
- ❌ Don't remove Docker socket access (needed for sub-container management)
- ✅ Configure via AIO web interface, not compose file
- ✅ Custom storage: Set `NEXTCLOUD_DATADIR` before first start

**Sub-Container Management**:
```bash
# View AIO-created containers
docker ps | grep nextcloud-aio-

# Don't manage sub-containers directly
# Use AIO interface at http://IP:8080
```

### Cloud Services: Immich (Split Deployment)

**Files**:
- `cloud/immich/compose.yml` - Main services (mini PC)
- `cloud/immich-ml/` - Documentation only (service runs on Unraid)

**Architecture**: Distributed deployment
- **Mini PC**: Web UI, API, microservices, PostgreSQL, Redis
- **Unraid**: ML container with NVIDIA P2000 GPU

**Critical Configuration**:
```yaml
# cloud/immich/compose.yml
services:
  immich-server:
    environment:
      - IMMICH_MACHINE_LEARNING_URL=http://192.168.1.x:3003  # Points to Unraid
    depends_on:
      - redis
      - database

  immich-microservices:
    environment:
      - IMMICH_MACHINE_LEARNING_URL=http://192.168.1.x:3003  # Same URL

  database:
    image: tensorchord/pgvecto-rs:pg14-v0.2.0  # PostgreSQL with vector extensions
    volumes:
      - ${DB_DATA_LOCATION}:/var/lib/postgresql/data:z
```

**When Modifying Immich**:
- ⚠️ Never add ML container to mini PC compose (runs on Unraid)
- ⚠️ `IMMICH_MACHINE_LEARNING_URL` must be reachable from mini PC
- ⚠️ Use `tensorchord/pgvecto-rs` image, not standard PostgreSQL
- ⚠️ Both server and microservices need ML URL
- ✅ Firewall on Unraid must allow mini PC access to port 3003
- ✅ Test connectivity: `curl http://UNRAID_IP:3003/ping`

**Split Architecture Decision Tree**:
```
Need to modify Immich?
├─ ML-related change? → Modify Unraid deployment (not compose) ⚠️
├─ Database change? → Mini PC compose, use pgvecto-rs image ✅
├─ Storage change? → Mini PC compose, update UPLOAD_LOCATION ✅
└─ Network issue? → Check ML URL and Unraid firewall ⚠️
```

## Common Implementation Patterns

### Adding a New Service

**Checklist**:
1. ✅ Choose appropriate stack (management/media/web/cloud)
2. ✅ Follow existing service template
3. ✅ Use environment variables for paths and secrets
4. ✅ Add `:z` to writable volume mounts
5. ✅ Include PUID/PGID/TZ environment variables
6. ✅ Add Watchtower label
7. ✅ Document in stack's README.md
8. ✅ Create `.env.example` entries with comments
9. ✅ Test with `docker compose config`
10. ✅ Update systemd service if new dependencies

**Template**:
```yaml
services:
  newservice:
    image: namespace/newservice:latest
    container_name: newservice
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true  # If internet-facing
    environment:
      - PUID=${PUID:-1000}
      - PGID=${PGID:-1000}
      - TZ=${TZ:-America/New_York}
      - SERVICE_PORT=${SERVICE_PORT:-8080}
      - SERVICE_PASSWORD=${SERVICE_PASSWORD}
    ports:
      - "${SERVICE_PORT:-8080}:8080"
    volumes:
      - ${APPDATA_PATH}/newservice:/config:z
      - /etc/localtime:/etc/localtime:ro
    networks:
      - web  # Choose appropriate network
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
    # If needs database
    depends_on:
      - database
```

### Database Services

**PostgreSQL Pattern**:
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
      - ${DB_DATA_LOCATION}:/var/lib/postgresql/data:z
    networks:
      - service_network
    # No external ports - internal access only
```

**Redis Pattern**:
```yaml
services:
  redis:
    image: redis:7-alpine
    container_name: service_redis
    restart: unless-stopped
    networks:
      - service_network
    # No volumes needed for cache-only usage
    # Add volume for persistence if required
```

### Health Checks

**HTTP Service**:
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

**Database**:
```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U ${DB_USERNAME:-postgres}"]
  interval: 10s
  timeout: 5s
  retries: 5
```

**Custom Script**:
```yaml
healthcheck:
  test: ["CMD", "/app/healthcheck.sh"]
  interval: 30s
  timeout: 10s
  retries: 3
```

### Resource Limits (When Needed)

```yaml
deploy:
  resources:
    limits:
      cpus: '2.0'
      memory: 4G
    reservations:
      cpus: '0.5'
      memory: 1G
```

**Use resource limits for**:
- Services with known memory leaks
- ML/processing workloads
- Services prone to resource exhaustion

**Don't over-constrain**:
- Let media transcoding use available resources
- Database services need memory for caching
- Avoid limits unless solving specific problems

## Troubleshooting Decision Trees

### Container Won't Start

```
Container fails to start?
├─ Check logs: docker logs <container>
│  ├─ Permission denied?
│  │  ├─ Missing :z tag? → Add to volume mount
│  │  ├─ Wrong PUID/PGID? → Check .env file
│  │  └─ Directory ownership? → chown -R 1000:1000 <path>
│  ├─ Port already in use?
│  │  └─ Check: netstat -tlnp | grep <port>
│  ├─ Missing environment variable?
│  │  └─ Check .env file and compose environment section
│  └─ Dependency not ready?
│     └─ Add depends_on or healthcheck
└─ Verify compose: docker compose config
```

### Service Can't Access Another Service

```
Service can't reach another service?
├─ Same stack?
│  ├─ Yes → Should work via service name
│  │  ├─ Check networks section
│  │  └─ Verify both on same bridge network
│  └─ No → Need explicit network connection
│     └─ Add both networks to both services
├─ Using host network mode?
│  └─ Use localhost or host IP, not service name
└─ External service (Immich-ML)?
   ├─ Check firewall on remote host
   ├─ Test: curl http://REMOTE_IP:PORT
   └─ Verify IMMICH_MACHINE_LEARNING_URL set
```

### Hardware Transcoding Not Working

```
QuickSync not working?
├─ Check device exists: ls -la /dev/dri/
│  ├─ No device? → Host issue, check CoreOS
│  └─ Device exists → Continue
├─ Check container access: docker exec <container> ls -la /dev/dri/
│  ├─ Permission denied? → Check compose devices section
│  └─ Device visible → Continue
├─ Jellyfin: Check LIBVA_DRIVER_NAME=iHD
├─ Test driver: docker exec jellyfin vainfo
└─ Check logs for hardware transcode attempts
```

### NFS Mount Issues

```
Media not accessible?
├─ Check mount: mount | grep nas-media
│  ├─ Not mounted?
│  │  ├─ Check /etc/fstab
│  │  ├─ Test: mount /mnt/nas-media
│  │  └─ Verify NFS server: showmount -e NFS_IP
│  └─ Mounted → Continue
├─ Check permissions: ls -la /mnt/nas-media
├─ In container: docker exec <container> ls -la /media
└─ Systemd service: Check RequiresMountsFor=/mnt/nas-media
```

## Expected Questions and Answers

### Q: Why Docker and not Podman on CoreOS?
**A**: Historical choice. This system was set up with Docker before Podman became standard. Migration would require:
- Recreating all systemd services
- Converting socket paths
- Testing all stacks
- Risk of downtime on remote system

**Decision**: Keep Docker. If starting fresh, Podman would be preferred.

### Q: Why separate management stack with different filename?
**A**: Historical. Management was the first stack created, using Docker's original `docker-compose.yml` convention. Other stacks adopted `compose.yml` (newer convention). Both work; maintaining consistency per-stack.

**Decision**: Don't rename. Both patterns are valid.

### Q: Why not use Docker Swarm or Kubernetes?
**A**:
- **Single-node deployment**: No clustering benefits
- **Simplicity**: Compose is sufficient for this scale
- **Remote hosting**: Complex orchestration increases failure points
- **Resource constraints**: Overhead not justified

**Decision**: Compose is appropriate for this use case.

### Q: Why QuickSync instead of GPU transcoding?
**A**:
- **Power efficiency**: ~15W vs. ~75W for GPU
- **24/7 operation**: Low power matters
- **Performance**: QuickSync handles 4-5 simultaneous 4K transcodes
- **Hardware available**: 12th gen Intel includes QuickSync

**Decision**: QuickSync is optimal for this hardware.

### Q: Why split Immich deployment?
**A**:
- **GPU location**: NVIDIA P2000 is in Unraid server
- **Storage**: Photos stored on mini PC (user access point)
- **Processing**: ML benefits from GPU, but most operations don't
- **Network**: Low latency LAN makes split viable

**Decision**: Optimize each component for its hardware strengths.

### Q: Why Nextcloud AIO instead of separate containers?
**A**:
- **Simplicity**: AIO handles all sub-containers automatically
- **Updates**: Coordinated updates through AIO interface
- **Configuration**: Integrated setup wizard
- **Maintenance**: Single point of management

**Decision**: AIO reduces operational complexity.

### Q: Should I add Traefik/Caddy instead of NginxProxyManager?
**A**: Evaluate trade-offs:
- **NPM**: GUI-based, easy SSL, existing setup
- **Traefik**: Automatic container discovery, label-based
- **Caddy**: Simple config, automatic HTTPS

**Decision Tree**:
- Keep NPM if it's working ✅
- Switch if specific feature needed ⚠️
- Avoid change for sake of change ❌

### Q: How to handle secrets management?
**A**: Current approach:
- `.env` files for secrets (git-ignored)
- Environment variables passed to containers
- No vault/secrets manager

**For production**: Consider Docker secrets or external vault. **For homelab**: Current approach is pragmatic.

## Critical Warnings

### ⚠️ NEVER Do These

1. **Modify systemd units without testing**
   - Remote system can't be recovered easily
   - Test in duplicate environment first

2. **Change network configuration**
   - WireGuard tunnel disruption = service outage
   - Tailscale changes = lost access
   - Always have out-of-band access plan

3. **Remove NFS mount while services running**
   - Will cause Plex/Jellyfin failures
   - May corrupt databases mid-write
   - Stop media stack first

4. **Modify Nextcloud sub-containers directly**
   - AIO will override changes
   - May break AIO management
   - Use AIO interface instead

5. **Add ML container to mini PC Immich stack**
   - Mini PC has no GPU
   - Breaks documented architecture
   - CPU-only ML is slow

6. **Hardcode secrets in compose files**
   - Security risk
   - Git history exposure
   - Use `.env` files

7. **Change Plex to bridge network**
   - Breaks device discovery
   - Breaks remote access
   - Breaks DLNA
   - Keep host mode

8. **Remove `:z` tags from existing volumes**
   - Will cause SELinux permission errors
   - Containers won't be able to write
   - Maintain existing patterns

### ⚠️ Changes Requiring Reboot

These changes require system reboot (due to immutable OS):
- Installing system packages via rpm-ostree
- Kernel module changes
- System-level configuration in /etc (some cases)

**Minimize reboots** by preferring containerized solutions.

### ⚠️ Changes Requiring Systemd Reload

```bash
# After modifying systemd service files
systemctl daemon-reload
systemctl restart docker-compose-<stack>.service
```

### ⚠️ Validation Steps Before Applying

**Always validate**:
```bash
# 1. Compose syntax
docker compose config

# 2. Environment variables resolved
docker compose config | grep -A 5 environment

# 3. Volume paths exist
ls -la ${APPDATA_PATH}/<service>

# 4. No port conflicts
netstat -tlnp | grep <port>

# 5. Dry run
docker compose up --dry-run
```

## Testing Workflow

**Before committing changes**:

1. **Validate syntax**:
   ```bash
   cd /srv/containers/<stack>
   docker compose config
   ```

2. **Check changes**:
   ```bash
   docker compose config | diff - <(docker compose config --file old-compose.yml)
   ```

3. **Test locally** (if possible):
   ```bash
   docker compose up -d <service>
   docker compose logs -f <service>
   ```

4. **Verify health**:
   ```bash
   docker compose ps
   docker inspect <container> | jq '.[0].State.Health'
   ```

5. **Check resource usage**:
   ```bash
   docker stats <container> --no-stream
   ```

6. **Test connectivity**:
   ```bash
   curl -I http://localhost:<port>
   ```

## Version Control Best Practices

**Git workflow**:
```bash
# 1. Create feature branch
git checkout -b add-<service>

# 2. Make changes
# ... edit compose files ...

# 3. Update documentation
# ... update README, .env.example ...

# 4. Commit with descriptive message
git add <stack>/compose.yml <stack>/.env.example <stack>/README.md
git commit -m "Add <service> to <stack> stack

- Purpose: <what it does>
- Port: <port>
- Network: <network>
- Dependencies: <if any>"

# 5. Test deployment
docker compose up -d

# 6. Verify functionality
docker compose ps
docker compose logs -f <service>

# 7. Push changes
git push origin add-<service>
```

**What NOT to commit**:
- `.env` files (secrets)
- `docker-compose.override.yml` (local overrides)
- Backup files (`*.bak`, `*~`)
- Log files

## Additional Resources

- **claude.md**: Technical reference for Claude AI support
- **DEPLOYMENT.md**: Deployment procedures and systemd setup
- **README.md**: Repository overview
- **Stack READMEs**: Service-specific documentation

**External Documentation**:
- [Docker Compose Spec](https://docs.docker.com/compose/compose-file/)
- [UBlue CoreOS](https://universal-blue.org/)
- [Immich Docs](https://immich.app/docs)
- [Nextcloud AIO](https://github.com/nextcloud/all-in-one)
- [Intel QuickSync](https://www.intel.com/content/www/us/en/architecture-and-technology/quick-sync-video/quick-sync-video-general.html)

## Quick Command Reference

```bash
# Validate compose file
docker compose config

# Start stack
docker compose up -d

# View logs
docker compose logs -f [service]

# Restart service
docker compose restart [service]

# Update images
docker compose pull
docker compose up -d

# Check status
docker compose ps

# Stop stack
docker compose down

# Remove and recreate
docker compose down
docker compose up -d --force-recreate

# Check systemd service
systemctl status docker-compose-<stack>.service
journalctl -u docker-compose-<stack>.service -f
```
