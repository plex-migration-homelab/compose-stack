# Compose-Stack Documentation Audit Prompt

Use this prompt to comprehensively audit and update the documentation for the compose-stack repository to accurately reflect current deployment reality.

---

## Repository Context

**Repository:** plex-migration-homelab/compose-stack
**Purpose:** Docker Compose configurations for homelab stack organized into media, web, and cloud stacks
**Deployment Path:** `/srv/containers/` (documented) vs `/home/user/compose-stack/` (current analysis location)
**Data Storage:** `/var/lib/containers/appdata/`

---

## 1. Documentation vs Reality Gaps

### CRITICAL: Media Stack Reality Check
**Issue:** The `media/compose.yml` contains ZERO actual services (`services: {}`), yet `media/README.md` documents as if Plex, Radarr, Sonarr, Lidarr, Prowlarr, and qBittorrent are configured.

**Audit Actions:**
- [ ] Document that media stack is intentionally a TEMPLATE, not a production configuration
- [ ] Clarify in media/README.md that services need to be manually configured
- [ ] Add example service definitions or link to templates
- [ ] Update "Accessing Services" section (lines 82-89) to note these are examples only
- [ ] Clearly state deployment status: "Template - Requires Configuration"

### Web Stack - Accurate Documentation
**Reality:** 4 production services configured
- Overseerr (port 5055)
- Wizarr (port 5690)
- Organizr (port 9983)
- Homepage (port 3000)

**Audit Actions:**
- [ ] Verify all service descriptions match current configurations
- [ ] Confirm port mappings are accurate
- [ ] Document Homepage's Docker socket access security implications

### Cloud Stack - Accurate Documentation
**Reality:** 6 production services configured
- Nextcloud (ports 8080, 8443) + MariaDB 10.11
- Immich (port 2283) + PostgreSQL (tensorchord/pgvecto-rs:pg14-v0.2.0) + Redis 7
- Collabora (port 9980)

**Audit Actions:**
- [ ] Verify database versions in documentation
- [ ] Confirm Immich uses pgvecto-rs variant (not standard PostgreSQL)
- [ ] Document Redis dependency for Immich

---

## 2. Current Compose File Structure Analysis

### Media Stack (`media/compose.yml`)
```yaml
services: {}  # EMPTY - Template only
networks:
  default:
    name: media_network
```
**Status:** No active services, contains commented example for Plex

### Web Stack (`web/compose.yml`)
**Services:**
1. **overseerr** - lscr.io/linuxserver/overseerr:latest
2. **wizarr** - ghcr.io/wizarrrr/wizarr:latest
3. **organizr** - lscr.io/linuxserver/organizr:latest
4. **homepage** - ghcr.io/gethomepage/homepage:latest
   - **SECURITY NOTE:** Mounts `/var/run/docker.sock:ro`

**Network:** web_network (custom bridge)

### Cloud Stack (`cloud/compose.yml`)
**Services:**
1. **nextcloud** - lscr.io/linuxserver/nextcloud:latest
   - Depends on: nextcloud-db
2. **nextcloud-db** - mariadb:10.11
3. **immich** - ghcr.io/immich-app/immich-server:release
   - Depends on: immich-db, immich-redis
4. **immich-db** - tensorchord/pgvecto-rs:pg14-v0.2.0
5. **immich-redis** - redis:7-alpine
6. **collabora** - collabora/code:latest
   - **SECURITY NOTE:** Requires `cap_add: MKNOD`

**Network:** cloud_network (custom bridge)

---

## 3. Environment Variables: Documented vs Actual Usage

### Media Stack Environment Variables

**In .env.example (13 lines):**
```bash
PUID=1000
PGID=1000
TZ=America/New_York
MEDIA_PATH=/path/to/media
TV_PATH=/path/to/media/tv
MOVIES_PATH=/path/to/media/movies
DOWNLOADS_PATH=/path/to/downloads
```

**Used in compose.yml:**
- NONE (services: {})

**Audit Actions:**
- [ ] Document that these variables are TEMPLATES for when services are added
- [ ] Clarify that MEDIA_PATH variables are not used until services are configured
- [ ] Add example of how to use these variables in service definitions

### Web Stack Environment Variables

**In .env.example (13 lines):**
```bash
PUID=1000
PGID=1000
TZ=America/New_York
# Commented optional port overrides:
# OVERSEERR_PORT=5055
# WIZARR_PORT=5690
# ORGANIZR_PORT=9983
# HOMEPAGE_PORT=3000
```

**Actually used in compose.yml:**
- PUID ✓ (all 4 services)
- PGID ✓ (all 4 services)
- TZ ✓ (all 4 services)
- Port variables ✗ (NOT USED - hardcoded in compose.yml)

**Audit Actions:**
- [ ] Remove misleading port override comments OR implement them in compose.yml
- [ ] Document that port changes require editing compose.yml directly
- [ ] Clarify which env vars are actually consumed

### Cloud Stack Environment Variables

**In .env.example (25 lines):**
```bash
PUID=1000
PGID=1000
TZ=America/New_York
MYSQL_ROOT_PASSWORD=changeme_root_password
MYSQL_PASSWORD=changeme_nextcloud_password
IMMICH_DB_PASSWORD=changeme_immich_password
NEXTCLOUD_DOMAIN=nextcloud\\.domain\\.com
COLLABORA_USERNAME=admin
COLLABORA_PASSWORD=changeme_collabora_password
# Commented optional port overrides:
# NEXTCLOUD_HTTP_PORT=8080
# NEXTCLOUD_HTTPS_PORT=8443
# IMMICH_PORT=2283
# COLLABORA_PORT=9980
```

**Actually used in compose.yml:**
- PUID ✓ (nextcloud, immich, collabora)
- PGID ✓ (nextcloud, immich, collabora)
- TZ ✓ (nextcloud, immich, collabora)
- MYSQL_ROOT_PASSWORD ✓ (nextcloud-db)
- MYSQL_PASSWORD ✓ (nextcloud-db as MYSQL_PASSWORD)
- IMMICH_DB_PASSWORD ✓ (immich, immich-db as POSTGRES_PASSWORD)
- NEXTCLOUD_DOMAIN ✓ (collabora as `domain`)
- COLLABORA_USERNAME ✓ (collabora as `username`)
- COLLABORA_PASSWORD ✓ (collabora as `password`)
- Port variables ✗ (NOT USED - hardcoded in compose.yml)

**Missing from .env.example but hardcoded:**
- DB_HOSTNAME=immich-db
- DB_USERNAME=immich
- DB_DATABASE_NAME=immich
- REDIS_HOSTNAME=immich-redis
- MYSQL_DATABASE=nextcloud
- MYSQL_USER=nextcloud
- POSTGRES_USER=immich
- POSTGRES_DB=immich

**Audit Actions:**
- [ ] Remove misleading port override comments OR implement them
- [ ] Document hardcoded database connection parameters
- [ ] Add security warning about changing default passwords
- [ ] Document NEXTCLOUD_DOMAIN regex format requirements

---

## 4. Deprecated or Outdated Service Configurations

### Image Version Analysis

**Potentially Outdated:**
- [ ] **Immich-db:** Uses `tensorchord/pgvecto-rs:pg14-v0.2.0` (pinned version from 2023)
  - **Action:** Check if newer pgvecto-rs versions are available and compatible
  - **Risk:** Vector extension may be outdated for newer Immich features

- [ ] **MariaDB:** Uses `mariadb:10.11` (specific minor version)
  - **Action:** Verify if 10.11 is still supported or if 11.x is recommended
  - **LTS Status:** Check MariaDB LTS schedule

**Using Latest Tags (Update Risk):**
- All other services use `:latest` or `:release` tags
- **Action:** Document that "latest" may cause breaking changes on updates
- **Recommendation:** Consider pinning major versions

### Deprecated Patterns

**Audit Actions:**
- [ ] Verify LinuxServer.io images haven't changed repository locations
- [ ] Check if `lscr.io/linuxserver/*` is still the canonical registry
- [ ] Confirm `ghcr.io` images are still maintained
- [ ] Review Docker Compose v2 syntax compatibility (currently correct)

---

## 5. Systemd Service Integration Patterns

**Current State:** NO actual systemd files in repository

**Documentation Location:** DEPLOYMENT.md lines 68-120

**Documented Pattern:**
```ini
[Unit]
Description=Docker Compose [Stack] Stack
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/srv/containers/[stack]
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```

**Audit Actions:**
- [ ] Create actual systemd service files in repository (e.g., `systemd/` directory)
- [ ] Add installation script for systemd files
- [ ] Document alternative: systemd user services vs system services
- [ ] Add troubleshooting for `systemctl` permission issues
- [ ] Document `TimeoutStartSec=0` rationale (why unlimited timeout?)
- [ ] Consider adding `Restart=on-failure` for reliability
- [ ] Document why `Type=oneshot` instead of `Type=forking`

**Missing from Systemd Documentation:**
- [ ] How to view service logs: `journalctl -u docker-compose-media.service -f`
- [ ] How to check service status after reboot
- [ ] Dependencies between stacks (should web stack wait for media?)
- [ ] Graceful shutdown order for database-backed services

---

## 6. Undocumented Troubleshooting Scenarios

### Missing Troubleshooting Sections

**Database Issues:**
- [ ] Nextcloud database connection failures on first start
- [ ] MariaDB data corruption recovery
- [ ] PostgreSQL pgvecto-rs extension initialization failures
- [ ] Immich database migration failures between versions

**Permission Problems:**
- [ ] PUID/PGID mismatch with existing data
- [ ] `/var/lib/containers/appdata` ownership issues
- [ ] Docker socket permission denied for Homepage
- [ ] SELinux/AppArmor blocking volume mounts

**Network Issues:**
- [ ] Services can't communicate across custom bridge networks
- [ ] Port conflicts with existing services
- [ ] IPv6 connectivity problems
- [ ] DNS resolution failures within containers

**Service-Specific:**
- [ ] Homepage can't connect to Docker socket
- [ ] Collabora refuses connection from Nextcloud
- [ ] Nextcloud "trusted domain" errors
- [ ] Overseerr can't reach Radarr/Sonarr (because media stack is empty)
- [ ] Immich mobile app connection timeouts

**Resource Constraints:**
- [ ] Out of disk space in `/var/lib/containers`
- [ ] Memory limits causing OOM kills
- [ ] CPU throttling affecting Immich machine learning

**Update/Migration Issues:**
- [ ] Breaking changes when pulling `:latest` images
- [ ] Database schema migrations requiring manual intervention
- [ ] Configuration format changes between versions

---

## 7. Actual Deployment Paths and File Locations

### Documented Locations

**From DEPLOYMENT.md and README.md:**
```
/srv/containers/              # Repository clone location
├── media/
│   ├── compose.yml
│   └── .env
├── web/
│   ├── compose.yml
│   └── .env
└── cloud/
    ├── compose.yml
    └── .env

/var/lib/containers/appdata/  # Application data
├── overseerr/
├── wizarr/
├── organizr/
├── homepage/
├── nextcloud/
│   ├── config/
│   ├── data/
│   └── db/
└── immich/
    ├── upload/
    ├── library/
    └── db/

/etc/systemd/system/          # Systemd service files
├── docker-compose-media.service
├── docker-compose-web.service
└── docker-compose-cloud.service
```

### Audit Actions

**Verify Documentation Covers:**
- [ ] Why `/srv/containers` vs `/opt` or `/home`?
- [ ] Filesystem requirements (ext4, xfs, btrfs compatibility)
- [ ] Disk space recommendations per service
- [ ] Backup paths and exclusions
- [ ] What happens if paths are changed?

**homelab-setup Integration:**
- [ ] **CRITICAL:** User mentioned "homelab-setup" but no references found
- [ ] Does a separate homelab-setup repository/script exist?
- [ ] Should this repository integrate with external setup tools?
- [ ] Document the relationship (if any) between compose-stack and homelab-setup

**Missing Path Documentation:**
- [ ] Docker socket location: `/var/run/docker.sock`
- [ ] Log locations for troubleshooting
- [ ] Where to find service-specific logs within appdata
- [ ] Temporary file locations
- [ ] Download cache locations

---

## 8. Security Considerations for Expose/Port Configurations

### Current Port Exposure Analysis

**Web Stack Ports (All on 0.0.0.0):**
```yaml
- "5055:5055"   # Overseerr
- "5690:5690"   # Wizarr
- "9983:80"     # Organizr
- "3000:3000"   # Homepage
```
**Risk:** Services exposed to all interfaces, no localhost-only binding

**Cloud Stack Ports (All on 0.0.0.0):**
```yaml
- "8080:80"     # Nextcloud HTTP
- "8443:443"    # Nextcloud HTTPS
- "2283:3001"   # Immich (note: external port ≠ internal port)
- "9980:9980"   # Collabora
```
**Risk:** Database services behind apps, but no authentication on local network

### Security Audit Actions

**Port Binding Security:**
- [ ] Document localhost-only binding option: `"127.0.0.1:5055:5055"`
- [ ] Explain when to use `0.0.0.0` vs `127.0.0.1`
- [ ] Add firewall configuration guidance (ufw/firewalld examples)
- [ ] Document reverse proxy requirement for external access

**Docker Socket Exposure (Homepage):**
```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:ro
```
- [ ] **HIGH RISK:** Document security implications
- [ ] Explain that Homepage has full read access to Docker
- [ ] Consider alternatives: Docker socket proxy
- [ ] Document that compromised Homepage = compromised host

**Collabora Capabilities:**
```yaml
cap_add:
  - MKNOD
```
- [ ] Document why MKNOD capability is required
- [ ] Explain security implications of Linux capabilities
- [ ] Verify if still needed in latest Collabora versions

**Database Exposure:**
- [ ] Document that databases are NOT exposed externally (good)
- [ ] Explain service-to-service communication via custom networks
- [ ] Security implications of network isolation

**Secrets Management:**
- [ ] `.env` files contain plaintext passwords
- [ ] Document that `.env` must never be committed (already in .gitignore)
- [ ] Consider Docker secrets or external secret management
- [ ] Document `.env` file permissions: `chmod 600 .env`

**Missing Security Documentation:**
- [ ] HTTPS/SSL certificate management
- [ ] Nextcloud trusted_domains configuration
- [ ] Fail2ban integration for web services
- [ ] Rate limiting recommendations
- [ ] Security headers for reverse proxy
- [ ] Two-factor authentication setup guides

---

## 9. Breaking Changes and Version Incompatibilities

### Known Version Constraints

**Immich-specific:**
- Immich server version MUST match database schema
- pgvecto-rs version tied to Immich version
- **Action:** Document Immich upgrade procedure (backup DB first!)
- **Action:** Pin Immich to major version or document rollback procedure

**Nextcloud:**
- Nextcloud upgrades must be sequential (can't skip major versions)
- Apps may break between major versions
- **Action:** Document Nextcloud upgrade path (23 → 24 → 25, etc.)
- **Action:** Document occ upgrade command sequence

**MariaDB → Nextcloud Compatibility:**
- Nextcloud 28+ requires MariaDB 10.6+
- Current: MariaDB 10.11 ✓ Compatible
- **Action:** Document MariaDB upgrade constraints

**PostgreSQL → Immich Compatibility:**
- pgvecto-rs extension version matters
- PostgreSQL 14 currently used
- **Action:** Document when to upgrade PostgreSQL version

### Docker Compose Syntax Changes

**Current:** Using Compose v2 syntax (correct)
- `compose.yml` (not `docker-compose.yml`)
- `docker compose` (not `docker-compose`)

**Audit Actions:**
- [ ] Document minimum Docker Compose version (v2.20+ per DEPLOYMENT.md)
- [ ] Explain why v2 syntax is used
- [ ] Add migration notes for users on v1

### Image Breaking Changes

**Services on `:latest` tag:**
- Overseerr, Wizarr, Organizr, Homepage, Nextcloud, Collabora
- **Risk:** Automatic updates may break functionality

**Audit Actions:**
- [ ] Document semantic versioning strategy
- [ ] Recommend testing updates in dev environment
- [ ] Add rollback instructions for each service
- [ ] Document how to pin to specific versions

**Known Breaking Changes to Document:**
- [ ] Immich v1.90+ changed API authentication
- [ ] Nextcloud 27→28 changed OIDC integration
- [ ] Homepage 0.8→0.9 changed config format
- [ ] Collabora CODE vs Collabora Online differences

---

## 10. Docker vs Podman Compatibility

### Current Status: Docker-Only

**Evidence:**
- All documentation uses `docker compose` commands
- systemd services call `/usr/bin/docker compose`
- No Podman-specific configuration files
- Docker socket mounted for Homepage: `/var/run/docker.sock`

### Audit Actions

**Document Docker-Specific Features:**
- [ ] Explicitly state "Docker required" in prerequisites
- [ ] Docker socket mount incompatible with Podman rootless
- [ ] Custom bridge networks work differently in Podman

**Podman Compatibility Gaps:**
- [ ] `docker-compose.yml` → `podman-compose` differences
- [ ] SELinux contexts (`:z` or `:Z` volume flags needed)
- [ ] Socket location: `/run/user/1000/podman/podman.sock` vs `/var/run/docker.sock`
- [ ] `network_mode: host` behaves differently
- [ ] Systemd integration uses `podman generate systemd` instead

**If Adding Podman Support:**
- [ ] Create separate `podman-compose.yml` files
- [ ] Document rootless vs rootful Podman deployment
- [ ] Add Podman-specific systemd service files
- [ ] Test all services with Podman 4.0+
- [ ] Document Podman network creation commands
- [ ] Update .gitignore for Podman files

**Alternative: Podman with Docker Compatibility:**
- [ ] Document `podman-docker` package installation
- [ ] Socket emulation: `systemctl --user enable podman.socket`
- [ ] Aliasing `docker` to `podman`
- [ ] Known compatibility issues

---

## Comprehensive Documentation Rewrite Checklist

### README.md Updates
- [ ] Add deployment status badges (Media: Template, Web: Production, Cloud: Production)
- [ ] Clarify media stack is a template requiring manual configuration
- [ ] Add "Quick Links" section to web UI ports
- [ ] Document actual vs example configurations
- [ ] Add architecture diagram showing service relationships
- [ ] Link to security best practices
- [ ] Add changelog/version history section

### DEPLOYMENT.md Updates
- [ ] Add actual systemd service files (not just templates)
- [ ] Expand troubleshooting section with scenarios from #6
- [ ] Add security hardening section
- [ ] Document backup testing procedures
- [ ] Add disaster recovery procedures
- [ ] Document monitoring with Prometheus/Grafana integration
- [ ] Add upgrade procedures for each service

### Stack-Specific README Updates

**media/README.md:**
- [ ] **CRITICAL:** Add prominent "TEMPLATE ONLY" notice
- [ ] Move example services to separate examples/ directory
- [ ] Add step-by-step guide for adding first service
- [ ] Document integration between services (Overseerr → Radarr, etc.)
- [ ] Remove "Accessing Services" section or mark as examples

**web/README.md:**
- [ ] Add Homepage Docker socket security warning
- [ ] Document how to configure Homepage widgets
- [ ] Add integration examples with media/cloud stacks
- [ ] Expand troubleshooting section

**cloud/README.md:**
- [ ] Document Nextcloud initial setup steps in detail
- [ ] Add Collabora integration troubleshooting
- [ ] Document Immich machine learning configuration
- [ ] Add performance tuning section for large deployments
- [ ] Document external access via reverse proxy
- [ ] Add GDPR/privacy considerations for cloud services

### New Documentation Needed
- [ ] **SECURITY.md:** Comprehensive security guide
- [ ] **UPGRADE.md:** Service-specific upgrade procedures
- [ ] **TROUBLESHOOTING.md:** Centralized troubleshooting guide
- [ ] **EXAMPLES.md:** Example service configurations for media stack
- [ ] **ARCHITECTURE.md:** System architecture and design decisions
- [ ] **FAQ.md:** Frequently asked questions
- [ ] **CONTRIBUTING.md:** How to contribute improvements

### .env.example Updates
- [ ] Remove unused port override variables OR implement them
- [ ] Add security warnings for default passwords
- [ ] Document which variables are required vs optional
- [ ] Add examples for different timezones
- [ ] Document NEXTCLOUD_DOMAIN regex format
- [ ] Add validation regex for each variable

### Code/Configuration Improvements
- [ ] Add healthchecks to all services
- [ ] Implement environment variable port overrides (if desired)
- [ ] Add resource limits (memory, CPU) to compose files
- [ ] Create actual systemd service files in `systemd/` directory
- [ ] Add Makefile or setup script for common operations
- [ ] Consider adding `.env.template` with comprehensive comments

---

## Priority Audit Order

### 🔴 CRITICAL (Do First)
1. Fix media stack documentation mismatch
2. Document Docker socket security risk (Homepage)
3. Remove or implement unused environment variables
4. Add security warnings for default passwords
5. Create actual systemd service files

### 🟡 HIGH PRIORITY
6. Expand troubleshooting documentation
7. Document breaking changes and upgrade procedures
8. Add version pinning recommendations
9. Document backup and restore procedures
10. Add security hardening guide

### 🟢 MEDIUM PRIORITY
11. Create architecture documentation
12. Add monitoring integration guide
13. Document reverse proxy setup
14. Create example configurations
15. Add FAQ section

### 🔵 LOW PRIORITY
16. Add Podman compatibility notes
17. Create contributing guidelines
18. Add performance tuning guides
19. Create video tutorials/screenshots
20. Add community resources links

---

## Output Format for Audit

For each issue found, document:

```markdown
### Issue: [Title]
**Severity:** Critical | High | Medium | Low
**Location:** [File:Line or Section]
**Current State:** [What currently exists]
**Expected State:** [What should exist]
**Impact:** [User impact of the discrepancy]
**Recommended Fix:** [Specific changes needed]
**Testing:** [How to verify the fix]
```

---

## Validation Checklist

After completing the audit, verify:

- [ ] All documented services exist in compose files
- [ ] All compose file services are documented
- [ ] All .env.example variables are actually used
- [ ] All compose.yml variables are in .env.example
- [ ] Port numbers match between docs and compose files
- [ ] Image versions are current and documented
- [ ] Security warnings are present for risky configurations
- [ ] Troubleshooting covers common failure scenarios
- [ ] Backup procedures are tested and documented
- [ ] Upgrade paths are clearly documented
- [ ] Links between documents are valid
- [ ] Code examples are tested and working

---

## Additional Context

**Repository Age:** Recent (created from initial commit)
**Last Major Update:** Add Docker Compose homelab structure (commit 66d9c98)
**Docker Compose Version Required:** v2.20+
**Docker Engine Version Required:** 24.0+
**Primary Use Case:** Homelab self-hosting
**Target Audience:** Technical users comfortable with Docker and Linux

---

## Questions to Resolve During Audit

1. **homelab-setup Integration:** Does a separate setup repository exist? Should it?
2. **Media Stack Intent:** Is the empty media stack intentional as a template?
3. **Port Variables:** Should unused port override variables be implemented or removed?
4. **Podman Support:** Is Podman compatibility a goal for this project?
5. **Systemd Files:** Should actual systemd files be included in the repository?
6. **Version Pinning:** Should images use `:latest` or pinned versions?
7. **Monitoring:** Should monitoring stack (Prometheus, Grafana) be included?
8. **Reverse Proxy:** Should a reverse proxy stack (Traefik, Nginx) be included?
9. **Backup Automation:** Should backup scripts be included or remain manual?
10. **Multi-host:** Is multi-host deployment (Docker Swarm, Kubernetes) in scope?

---

**Generated:** 2025-11-19
**For:** plex-migration-homelab/compose-stack repository audit
**Purpose:** Comprehensive documentation alignment with deployment reality
