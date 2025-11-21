# AI Agent Instructions: Docker Compose Homelab Stack

You are an AI assistant working with a Fedora CoreOS Docker Compose homelab stack. Follow these instructions precisely.

---

## CRITICAL SYSTEM CONTEXT

This system runs **Fedora CoreOS** with **SELinux in enforcing mode**. Every decision you make must account for:

1. **SELinux is MANDATORY** - Containers cannot write to bind mounts without proper SELinux labels
2. **System is remotely hosted** - Boot failures require physical intervention by Marcus
3. **Immutable OS** - Cannot install packages traditionally (uses rpm-ostree)
4. **Docker runtime** - NOT Podman (despite CoreOS convention)
5. **Systemd-managed services** - Stacks controlled by systemd, not manual commands

**Hardware:**
- NAB9 Mini PC: Intel i5 12th gen (QuickSync), 32GB RAM, local appdata storage
- Unraid Server: ZFS/NFS media storage, hosts Immich-ML with NVIDIA P2000 GPU
- Digital Ocean VPS: NginxProxyManager, WireGuard tunnel to mini PC

**Network:**
- WireGuard: VPS ↔ Mini PC (public service access)
- Tailscale: Internal administrative access
- NFS: Unraid → Mini PC (`/mnt/nas-media`)

---

## MANDATORY SELINUX RULES

When you write or modify ANY volume mount in a Docker Compose file, you MUST follow these rules:

### Rule 1: Writable Bind Mounts REQUIRE `:z`

**WHEN** you mount a host directory that containers need to **write** to:
**THEN** you MUST add the `:z` SELinux context tag

```yaml
# ✅ CORRECT - Container can write
volumes:
  - ${APPDATA_PATH}/plex:/config:z
  - ${DB_DATA_LOCATION}:/var/lib/postgresql/data:z
  - ${UPLOAD_LOCATION}:/uploads:z

# ❌ WRONG - Will fail with "permission denied"
volumes:
  - ${APPDATA_PATH}/plex:/config
  - ${DB_DATA_LOCATION}:/var/lib/postgresql/data
```

**WHY:** The `:z` flag tells Docker to relabel the directory with `svirt_sandbox_file_t`, allowing container write access under SELinux.

### Rule 2: Read-Only System Files Use ONLY `:ro`

**WHEN** you mount system files or read-only resources:
**THEN** you MUST use `:ro` WITHOUT `:z`

```yaml
# ✅ CORRECT - Read-only system resources
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:ro
  - /etc/localtime:/etc/localtime:ro
  - /mnt/nas-media:/media:ro

# ❌ WRONG - Never add :z to these
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:z
  - /etc/localtime:/etc/localtime:z
```

### Rule 2a: Docker Socket Access Requires Special SELinux Context

**WHEN** a container needs to **write** to the Docker socket (Portainer, Watchtower, etc.):
**THEN** you MUST use `security_opt: - label=type:container_runtime_t`

```yaml
# ✅ CORRECT - Container can manage Docker socket
services:
  portainer:
    image: portainer/portainer-ce:latest
    user: "0:0"                              # Numeric UID:GID (avoids /etc/passwd lookups)
    security_opt:
      - no-new-privileges:true
      - label=type:container_runtime_t       # Grants Docker socket access via SELinux
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:z  # Use :z with container_runtime_t
      - ${APPDATA_PATH}/portainer:/data:z

# ❌ WRONG - Contradictory SELinux configuration
services:
  portainer:
    security_opt:
      - label=disable                        # Disables ALL SELinux labeling
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:z  # :z has no effect with label=disable
```

**WHY:** The `container_runtime_t` SELinux type gives the container the appropriate security context to interact with the Docker socket while maintaining SELinux enforcement. Using `label=disable` with `:z` flags creates a contradiction where SELinux labeling is disabled but you're trying to apply labels.

**Alternative approaches (in order of preference):**
1. **Use `label=type:container_runtime_t`** (RECOMMENDED) - Maintains SELinux enforcement with proper context
2. **Enable SELinux boolean** - `setsebool -P container_manage_cgroup on` (system-wide change)
3. **Use `label=disable`** - Only if absolutely necessary, and remove all `:z` flags (least secure)

### Rule 3: Named Volumes Need NO Tags

**WHEN** you use Docker-managed named volumes:
**THEN** you MUST NOT add any SELinux tags

```yaml
# ✅ CORRECT - Named volumes
volumes:
  - postgres_data:/var/lib/postgresql/data
  - model-cache:/cache

volumes:
  postgres_data:
    name: service_postgres_data
```

### SELinux Quick Decision Table

| What You're Mounting | Tag Required | Security Context | Example |
|---------------------|-------------|-----------------|---------|
| App config directory | `:z` | N/A | `${APPDATA_PATH}/service:/config:z` |
| Database directory | `:z` | N/A | `${DB_DATA_LOCATION}:/var/lib/postgresql/data:z` |
| Upload directory | `:z` | N/A | `${UPLOAD_LOCATION}:/uploads:z` |
| Cache directory (writable) | `:z` | N/A | `${APPDATA_PATH}/service/cache:/cache:z` |
| Docker socket (read-only) | `:ro` ONLY | N/A | `/var/run/docker.sock:/var/run/docker.sock:ro` |
| Docker socket (write) | `:z` | `label=type:container_runtime_t` | See Rule 2a |
| System time files | `:ro` ONLY | N/A | `/etc/localtime:/etc/localtime:ro` |
| NFS read-only media | `:ro` ONLY | N/A | `/mnt/nas-media:/media:ro` |
| Named Docker volume | NONE | N/A | `volume_name:/path` |

---

## STACK-SPECIFIC INSTRUCTIONS

### Management Stack (`management/docker-compose.yml`)

**File naming:** This stack uses `docker-compose.yml` (NOT `compose.yml`) - historical convention, do NOT change.

**Services:** Portainer (9000, 9443), Watchtower (auto-updates), Tautulli (8181)

**WHEN** modifying this stack:
- Portainer and Watchtower need Docker socket **write** access
- MUST use `user: "0:0"` (numeric UID:GID)
- MUST use `security_opt: - label=type:container_runtime_t` for SELinux context
- Docker socket MUST use `:z`: `/var/run/docker.sock:/var/run/docker.sock:z`
- All appdata mounts MUST use `:z`: `${APPDATA_PATH}/portainer:/data:z`
- Watchtower MUST have `WATCHTOWER_LABEL_ENABLE=true` environment variable
- All services MUST include label: `com.centurylinklabs.watchtower.enable=true`

**NEVER:**
- Use `label=disable` with `:z` volume flags (contradictory configuration)
- Use `:ro` on Docker socket for Portainer/Watchtower (they need write access)

### Media Stack (`media/compose.yml`)

**Services:** Plex (32400), Jellyfin (8096)

**WHEN** modifying this stack:
- Plex MUST use `network_mode: host` (NEVER change to bridge - breaks discovery)
- Jellyfin MUST have `LIBVA_DRIVER_NAME=iHD` environment variable (12th gen Intel QuickSync)
- Both MUST have device access: `devices: [/dev/dri:/dev/dri]`
- All appdata MUST use `:z`: `${APPDATA_PATH}/plex:/config:z`
- NFS media MUST use `:ro` only: `/mnt/nas-media:/media:ro` (no `:z`)
- Systemd service MUST include: `RequiresMountsFor=/mnt/nas-media`

**NEVER:**
- Change Plex to bridge network
- Remove `/dev/dri` device mount
- Remove `LIBVA_DRIVER_NAME=iHD` from Jellyfin

### Web Stack (`web/compose.yml`)

**Services:** NginxProxyManager (80, 443, 81), Overseerr (5055), Wizarr (5690), Homepage (3000)

**WHEN** modifying this stack:
- All services MUST be on `web` bridge network
- All appdata MUST use `:z`: `${APPDATA_PATH}/service:/config:z`
- Homepage needs Docker socket **read-only** access: `/var/run/docker.sock:/var/run/docker.sock:ro`
- If Homepage has socket permission issues, use `security_opt: - label=type:container_runtime_t`
- Services are accessed via WireGuard tunnel from VPS

### Nextcloud AIO Stack (`cloud/nextcloud/compose.yml`)

**Architecture:** Single mastercontainer (NOT multi-container stack)

**WHEN** modifying this stack:
- There is ONLY ONE container: `nextcloud-aio-mastercontainer`
- Mastercontainer auto-creates sub-containers (`nextcloud-aio-*`) via Docker API
- MUST have: `init: true`
- Needs Docker socket **write** access: `/var/run/docker.sock:/var/run/docker.sock:z`
- MUST use `security_opt: - label=type:container_runtime_t` for SELinux context
- MUST use named volume: `nextcloud_aio_mastercontainer:/mnt/docker-aio-config` (no tags)
- Port 8080 is the AIO admin interface

**NEVER:**
- Add PostgreSQL, Redis, or Apache containers to the compose file (AIO creates these)
- Modify `nextcloud-aio-*` sub-containers directly
- Change the mastercontainer volume name
- Remove Docker socket access

**IF** user asks to modify Nextcloud configuration:
**THEN** direct them to use AIO web interface at `http://IP:8080`

### Immich Stack (`cloud/immich/compose.yml`)

**Architecture:** Split deployment - main services on mini PC, ML on Unraid

**WHEN** modifying this stack:
- Server and microservices MUST have: `IMMICH_MACHINE_LEARNING_URL=http://UNRAID_IP:3003`
- All uploads MUST use `:z`: `${UPLOAD_LOCATION}:/usr/src/app/upload:z`
- Database MUST use `:z`: `${DB_DATA_LOCATION}:/var/lib/postgresql/data:z`
- System time MUST use `:ro`: `/etc/localtime:/etc/localtime:ro`

**NEVER:**
- Add `immich-machine-learning` container to this compose file (runs on Unraid)
- Remove `IMMICH_MACHINE_LEARNING_URL` from server or microservices

**IF** user asks about ML container:
**THEN** inform them it runs on Unraid server (port 3003), not in this compose stack

### Immich-ML (`cloud/immich-ml/`)

**Architecture:** Documentation only - actual service runs on Unraid via Docker UI

**WHEN** user asks about this directory:
- Contains setup documentation, NOT a compose stack for the mini PC
- Actual service deployed via Docker web UI on Unraid server
- Requires NVIDIA P2000 GPU (only available on Unraid)
- Must be accessible from mini PC on port 3003

**NEVER:**
- Deploy this compose file on the mini PC (no GPU available)
- Suggest moving ML to mini PC (would be CPU-only and very slow)

---

## STANDARD COMPOSE PATTERNS

### When Adding a New Service

**ALWAYS** include these elements:

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
      - ${APPDATA_PATH}/servicename:/config:z  # MUST have :z
      - /etc/localtime:/etc/localtime:ro        # MUST have :ro
    networks:
      - stack_network
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
```

**Checklist before committing:**
- [ ] All writable bind mounts have `:z`
- [ ] Read-only system files have `:ro` only
- [ ] PUID, PGID, TZ environment variables included
- [ ] Appropriate network assigned
- [ ] Watchtower label included
- [ ] Container name specified
- [ ] `restart: unless-stopped` (not `always`)

### When Adding a Database

```yaml
services:
  database:
    image: postgres:16-alpine
    container_name: service_postgres
    restart: unless-stopped
    environment:
      - POSTGRES_USER=${DB_USERNAME:-postgres}
      - POSTGRES_PASSWORD=${DB_PASSWORD}  # From .env
      - POSTGRES_DB=${DB_DATABASE_NAME:-servicedb}
    volumes:
      - ${DB_DATA_LOCATION}:/var/lib/postgresql/data:z  # MUST have :z
    networks:
      - service_network
    # No external ports - internal access only
```

**NEVER:**
- Hardcode database passwords (use `${DB_PASSWORD}` from `.env`)
- Expose database ports externally unless absolutely required
- Forget `:z` on data directory

---

## DECISION TREES

### When Container Fails with "Permission Denied"

```
1. Check container logs: docker logs <container_name>
2. IF error mentions /config, /data, or any mounted path:
   ├─ Check compose file volume mounts
   ├─ IF bind mount is missing :z tag:
   │  └─ ADD :z tag to the volume mount
   ├─ ELSE check SELinux context: ls -Z ${APPDATA_PATH}/<service>
   │  └─ IF context is not svirt_sandbox_file_t:
   │     └─ ADD :z tag to volume mount
   └─ ELSE check ownership: ls -la ${APPDATA_PATH}/<service>
      └─ IF owned by wrong user:
         └─ chown -R 1000:1000 ${APPDATA_PATH}/<service>
```

### When User Asks to Add a Service

```
1. Determine appropriate stack:
   ├─ Infrastructure/monitoring? → management/
   ├─ Media streaming? → media/
   ├─ Web application? → web/
   └─ Cloud/storage? → cloud/

2. Check for special requirements:
   ├─ Needs GPU? → Can only run on Unraid (mini PC has no GPU)
   ├─ Needs QuickSync? → media/ stack with /dev/dri device
   ├─ Needs Docker socket? → Must use :ro, never :z
   └─ Needs database? → Create in same stack

3. Write compose configuration:
   ├─ Use standard service template
   ├─ Add :z to all writable bind mounts
   ├─ Add :ro to system files
   ├─ Include PUID/PGID/TZ
   └─ Add to appropriate network

4. Create supporting files:
   ├─ Add variables to .env.example with comments
   ├─ Document in stack's README.md
   └─ Update systemd service if dependencies change

5. Validate before committing:
   └─ docker compose config (check syntax)
```

### When User Reports Service Not Working

```
1. Check service status:
   docker compose ps

2. IF container not running:
   ├─ Check logs: docker compose logs -f <service>
   ├─ IF permission denied → See permission decision tree above
   ├─ IF port conflict → Check: netstat -tlnp | grep <port>
   └─ IF dependency issue → Check depends_on and healthcheck

3. IF container running but not accessible:
   ├─ Check ports: docker compose ps
   ├─ Check network: docker network inspect <network>
   ├─ IF cross-stack communication needed:
   │  └─ Add both networks to both services
   └─ IF external service (Immich-ML):
      └─ Test: curl http://REMOTE_IP:PORT

4. IF QuickSync not working:
   ├─ Check device exists: ls -la /dev/dri/
   ├─ Check container access: docker exec <container> ls -la /dev/dri/
   ├─ Check LIBVA_DRIVER_NAME=iHD (Jellyfin)
   └─ Test: docker exec jellyfin vainfo
```

---

## ABSOLUTE RULES (NEVER VIOLATE)

### NEVER Do These (System Will Break):

1. **Remove `:z` from writable bind mounts**
   - Containers will fail with permission denied
   - SELinux will block all write operations

2. **Combine `label=disable` with `:z` volume flags**
   - Creates contradictory SELinux configuration
   - The `:z` flags become ineffective when SELinux labeling is disabled
   - Use `label=type:container_runtime_t` instead for Docker socket access

3. **Use `:ro` on Docker socket for containers that need write access**
   - Portainer, Watchtower, Nextcloud AIO need write access
   - They MUST use `:z` with `label=type:container_runtime_t` security context
   - Homepage only needs `:ro` (read-only access)

4. **Add `:z` to named volumes**
   - Docker manages these automatically
   - Adding :z will cause errors

5. **Change Plex to bridge network**
   - Breaks device discovery
   - Breaks remote access
   - Breaks DLNA

6. **Add ML container to mini PC Immich stack**
   - Mini PC has no GPU
   - Would be CPU-only (very slow)
   - Breaks documented architecture

7. **Modify Nextcloud sub-containers directly**
   - AIO will override changes
   - May break entire AIO system

8. **Modify systemd units without testing**
   - System is remote - cannot physically recover
   - Could lose access entirely

9. **Change network configuration**
   - WireGuard tunnel is critical for access
   - Tailscale is backup access
   - Loss of both = system unreachable

10. **Hardcode secrets in compose files**
    - Security risk
    - Git history exposure
    - Always use ${VAR} from .env

11. **Use `apt`, `dnf`, or `yum`**
    - Fedora CoreOS is immutable
    - Use rpm-ostree for system packages (requires reboot)
    - Prefer containerized solutions

### ALWAYS Do These:

1. **Validate compose syntax before committing**
   ```bash
   docker compose config
   ```

2. **Check for SELinux denials after changes**
   ```bash
   sudo ausearch -m avc -ts recent
   ```

3. **Use environment variables for paths and secrets**
   - `${APPDATA_PATH}/service:/config:z`
   - `${DB_PASSWORD}` not `password123`

4. **Include standard environment variables**
   - PUID=${PUID:-1000}
   - PGID=${PGID:-1000}
   - TZ=${TZ:-America/New_York}

5. **Document changes**
   - Update .env.example with new variables
   - Update stack README.md
   - Add comments explaining why

6. **Consider remote hosting constraint**
   - Can this change cause boot failure?
   - Can this change break systemd?
   - Is there a rollback plan?

7. **Respect systemd management**
   - Services controlled by systemd
   - Don't suggest manual `docker compose up`
   - Suggest: `systemctl restart docker-compose-<stack>.service`

---

## VALIDATION CHECKLIST

Before suggesting ANY changes to compose files, verify:

**SELinux Compliance:**
- [ ] All writable bind mounts have `:z`
- [ ] Docker socket (read-only) has `:ro` only (no `:z`)
- [ ] Docker socket (write) has `:z` AND `label=type:container_runtime_t`
- [ ] NEVER combine `label=disable` with `:z` flags (contradictory)
- [ ] System files have `:ro` only (no `:z`)
- [ ] NFS mounts have `:ro` only (no `:z`)
- [ ] Named volumes have no tags

**Security:**
- [ ] No hardcoded secrets
- [ ] Secrets use `${VAR}` from .env
- [ ] Internet-facing services have `no-new-privileges:true`
- [ ] Database ports not externally exposed

**Patterns:**
- [ ] PUID/PGID/TZ environment variables included
- [ ] Container name specified
- [ ] Appropriate network assigned
- [ ] Watchtower label included (unless explicitly disabled)
- [ ] `restart: unless-stopped` (not `always`)

**Stack-Specific:**
- [ ] Management stack: Uses `docker-compose.yml` filename
- [ ] Plex: Uses host network mode
- [ ] Jellyfin: Has `LIBVA_DRIVER_NAME=iHD`
- [ ] QuickSync services: Have `/dev/dri` device
- [ ] Nextcloud: Only mastercontainer in compose
- [ ] Immich: Has `IMMICH_MACHINE_LEARNING_URL` set
- [ ] Immich database: Uses `tensorchord/pgvecto-rs` image

**Testing:**
- [ ] `docker compose config` passes
- [ ] Volume paths exist on host
- [ ] No port conflicts
- [ ] Environment variables defined in .env.example
- [ ] Documentation updated

---

## QUICK REFERENCE

### File Locations
- **Repository:** `/srv/containers/`
- **Appdata:** `/var/lib/containers/appdata/`
- **NFS Media:** `/mnt/nas-media` (read-only)
- **Management:** `management/docker-compose.yml` (exception to naming)
- **Others:** `<stack>/compose.yml` (with symlink to `docker-compose.yml`)

### Common Environment Variables
```bash
PUID=1000
PGID=1000
TZ=America/New_York
APPDATA_PATH=/var/lib/containers/appdata
```

### Systemd Service Pattern
```ini
[Unit]
Description=Docker Compose [Stack] Stack
Requires=docker.service network-online.target
After=docker.service network-online.target
RequiresMountsFor=/mnt/nas-media  # If NFS needed

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

### Troubleshooting Commands
```bash
# Check SELinux denials
sudo ausearch -m avc -ts recent | grep denied

# Check SELinux context
ls -Z /var/lib/containers/appdata/<service>

# Validate compose
docker compose config

# Check container logs
docker compose logs -f <service>

# Check service status
systemctl status docker-compose-<stack>.service
```

---

## RESPONSE FORMAT

When suggesting changes to compose files:

1. **State what you're doing and why**
2. **Show the exact YAML to add/change**
3. **Highlight SELinux tags explicitly**
4. **Explain any deviations from patterns**
5. **Provide validation commands**
6. **Warn about potential issues**

Example:
```
I'll add Radarr to the web stack. This service needs:
- Writable appdata mount (requires :z tag)
- Read-only media mount (requires :ro only)
- Connection to web network for VPS access

Here's the configuration:

```yaml
radarr:
  image: lscr.io/linuxserver/radarr:latest
  container_name: radarr
  restart: unless-stopped
  environment:
    - PUID=${PUID:-1000}
    - PGID=${PGID:-1000}
    - TZ=${TZ:-America/New_York}
  ports:
    - "7878:7878"
  volumes:
    - ${APPDATA_PATH}/radarr:/config:z        # Writable - needs :z
    - /mnt/nas-media:/media:ro                 # Read-only NFS - :ro only
  networks:
    - web
  labels:
    - "com.centurylinklabs.watchtower.enable=true"
```

Add to .env.example:
```bash
# Radarr - Movie management
RADARR_PORT=7878
```

Validate with:
```bash
cd /srv/containers/web
docker compose config
```

After deploying, restart the systemd service:
```bash
systemctl restart docker-compose-web.service
```
```

---

**Remember:** Stability and SELinux compliance are non-negotiable. When in doubt, ask for clarification rather than making assumptions.
