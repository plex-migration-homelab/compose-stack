# Architecture Overview

This document provides a comprehensive overview of the Docker Compose homelab stack architecture, design decisions, and technical details.

## Table of Contents

- [System Architecture](#system-architecture)
- [Stack Organization](#stack-organization)
- [Networking](#networking)
- [Data Storage](#data-storage)
- [Security Model](#security-model)
- [Design Decisions](#design-decisions)

## System Architecture

### High-Level Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        Host System                          │
│                     (Linux with Docker)                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐  │
│  │  Media Stack  │  │   Web Stack   │  │  Cloud Stack  │  │
│  │               │  │               │  │               │  │
│  │  - Plex       │  │  - Overseerr  │  │  - Nextcloud  │  │
│  │  - Sonarr     │  │  - Wizarr     │  │  - Immich     │  │
│  │  - Radarr     │  │  - Organizr   │  │  - Collabora  │  │
│  │  - Prowlarr   │  │  - Homepage   │  │               │  │
│  │  - qBittorrent│  │               │  │               │  │
│  └───────────────┘  └───────────────┘  └───────────────┘  │
│         │                   │                   │          │
│         └───────────────────┴───────────────────┘          │
│                             │                              │
│                    ┌────────▼────────┐                     │
│                    │  Docker Engine  │                     │
│                    └────────┬────────┘                     │
│                             │                              │
│            ┌────────────────┴────────────────┐             │
│            │                                 │             │
│    ┌───────▼───────┐              ┌─────────▼────────┐    │
│    │  Data Storage │              │     Networks     │    │
│    │               │              │                  │    │
│    │  /srv/        │              │  - media_network │    │
│    │  /var/lib/    │              │  - web_network   │    │
│    │               │              │  - cloud_network │    │
│    └───────────────┘              └──────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Component Layers

1. **Host Operating System**
   - Linux-based system with systemd
   - Docker Engine and Docker Compose
   - Network and storage configuration

2. **Container Orchestration**
   - Docker Engine manages container lifecycle
   - Docker Compose manages multi-container applications
   - Systemd services for automatic startup

3. **Application Stacks**
   - Three independent stacks (media, web, cloud)
   - Each stack has its own network namespace
   - Services communicate within their stack

4. **Storage Layer**
   - Configuration in `/srv/containers/`
   - Application data in `/var/lib/containers/appdata/`
   - User data (media, files) in custom locations

## Stack Organization

### Directory Structure

```
/srv/containers/                    # Stack configurations
├── media/                          # Media management stack
│   ├── compose.yml                 # Service definitions
│   ├── docker-compose.yml          # Compatibility symlink
│   ├── .env.example                # Environment template
│   └── README.md                   # Stack documentation
├── web/                            # Web applications stack
│   ├── compose.yml
│   ├── docker-compose.yml
│   ├── .env.example
│   └── README.md
├── cloud/                          # Cloud services stack
│   ├── compose.yml
│   ├── docker-compose.yml
│   ├── .env.example
│   └── README.md
├── README.md                       # Main documentation
├── DEPLOYMENT.md                   # Deployment guide
├── ARCHITECTURE.md                 # This document
├── CONTRIBUTING.md                 # Contributor guidelines
├── FAQ.md                          # Frequently asked questions
└── CHANGELOG.md                    # Version history

/var/lib/containers/appdata/       # Application data
├── [service-name]/                # Per-service data directory
│   ├── config/                    # Configuration files
│   ├── data/                      # Application data
│   └── db/                        # Database files (if applicable)
```

### Stack Independence

Each stack operates independently:
- **Separate networks**: Isolated network namespaces
- **Independent lifecycle**: Start/stop without affecting others
- **Isolated failures**: Issues in one stack don't impact others
- **Flexible deployment**: Deploy only what you need

### Cross-Stack Communication

When services need to communicate across stacks:
1. Use Docker's internal DNS (service name resolution)
2. Configure external networks if needed
3. Use host networking for specific use cases
4. Implement reverse proxy for unified access

## Networking

### Network Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      Host Network                       │
│                    (Physical/Virtual)                   │
└────────────┬────────────────────────────┬───────────────┘
             │                            │
    ┌────────▼────────┐          ┌────────▼────────┐
    │   Port Mapping  │          │   Port Mapping  │
    │   (Published)   │          │   (Published)   │
    └────────┬────────┘          └────────┬────────┘
             │                            │
    ┌────────▼────────────────────────────▼────────┐
    │          Docker Bridge Networks              │
    ├──────────────────────────────────────────────┤
    │                                              │
    │  ┌──────────────┐  ┌──────────────┐         │
    │  │media_network │  │ web_network  │         │
    │  ├──────────────┤  ├──────────────┤         │
    │  │ Container A  │  │ Container X  │         │
    │  │ Container B  │  │ Container Y  │         │
    │  └──────────────┘  └──────────────┘         │
    │                                              │
    │  ┌──────────────┐                            │
    │  │cloud_network │                            │
    │  ├──────────────┤                            │
    │  │ Container 1  │                            │
    │  │ Container 2  │                            │
    │  │ Container 3  │                            │
    │  └──────────────┘                            │
    │                                              │
    └──────────────────────────────────────────────┘
```

### Network Types

1. **Bridge Networks** (Default)
   - `media_network`: Media stack services
   - `web_network`: Web application services
   - `cloud_network`: Cloud service stack
   - Provides isolation and DNS resolution
   - Containers communicate via service names

2. **Host Network** (Optional)
   - Direct host network access
   - No port mapping needed
   - Used for services requiring host network features
   - Less secure, use sparingly

3. **Port Publishing**
   - Maps container ports to host ports
   - Format: `"HOST_PORT:CONTAINER_PORT"`
   - Allows external access to services
   - Configure firewall rules accordingly

### DNS Resolution

- **Within stack**: Use service name (e.g., `http://nextcloud:80`)
- **Across stacks**: Use external networking or host IPs
- **External access**: Use host IP with published ports

## Data Storage

### Storage Architecture

```
┌────────────────────────────────────────────────────────┐
│                    Host Filesystem                     │
├────────────────────────────────────────────────────────┤
│                                                        │
│  Configuration Storage        Application Data        │
│  /srv/containers/            /var/lib/containers/     │
│  ├── compose.yml             ├── appdata/             │
│  ├── .env                    │   ├── service1/        │
│  └── README.md               │   │   ├── config/      │
│                              │   │   ├── data/        │
│                              │   │   └── db/          │
│                              │   └── service2/        │
│                              │                        │
│  User Data (Custom)                                   │
│  /path/to/media/                                      │
│  ├── movies/                                          │
│  ├── tv/                                              │
│  └── music/                                           │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Volume Mounts

**Bind Mounts** (Used throughout):
- Maps host directory to container path
- Format: `"HOST_PATH:CONTAINER_PATH[:OPTIONS]"`
- Allows direct file access from host
- Survives container recreation

**Named Volumes** (Alternative):
- Docker-managed volumes
- Better for databases
- Portable across systems
- Less direct host access

### Data Persistence Strategy

1. **Configuration Files**: `/srv/containers/`
   - Git-tracked for version control
   - Easy to backup and restore
   - Portable across systems

2. **Application Data**: `/var/lib/containers/appdata/`
   - Service-specific subdirectories
   - Preserved across container updates
   - Regular backup recommended

3. **User Content**: Custom paths
   - Media libraries
   - Photo/video storage
   - Document repositories

### Permissions Model

All services use PUID/PGID for consistent permissions:
- Set in `.env` file
- Matches host user ID/group ID
- Ensures proper file access
- Simplifies permission management

## Security Model

### Layered Security

```
┌─────────────────────────────────────────┐
│        External Access Layer            │
│  - Firewall rules                       │
│  - Reverse proxy (recommended)          │
│  - SSL/TLS termination                  │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│      Application Security Layer         │
│  - Service authentication               │
│  - Role-based access control            │
│  - API keys and tokens                  │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│       Container Isolation Layer         │
│  - Separate network namespaces          │
│  - Limited capabilities                 │
│  - Read-only filesystems (where possible)│
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│        Host Security Layer              │
│  - OS-level security                    │
│  - File permissions (PUID/PGID)         │
│  - Docker daemon security               │
└─────────────────────────────────────────┘
```

### Security Best Practices

1. **Network Security**
   - Use reverse proxy for external access
   - Enable SSL/TLS with valid certificates
   - Configure firewall rules (UFW, iptables)
   - Consider VPN for remote access

2. **Authentication**
   - Change default passwords immediately
   - Use strong, unique passwords
   - Enable 2FA where available
   - Implement SSO if possible

3. **Container Security**
   - Keep images updated
   - Use official or trusted images
   - Limit container capabilities
   - Run as non-root where possible

4. **Data Security**
   - Encrypt sensitive data
   - Regular backups
   - Secure backup storage
   - Test restore procedures

5. **Access Control**
   - Principle of least privilege
   - Separate admin accounts
   - Audit access logs
   - Regular permission reviews

### Secrets Management

- **Never commit secrets to git**
- Use `.env` files (git-ignored)
- Consider Docker secrets for production
- Rotate passwords regularly
- Use key management systems for scaling

## Design Decisions

### Why Docker Compose?

- **Simplicity**: Easy to understand and maintain
- **Portability**: Runs on any Docker-capable system
- **Flexibility**: Easy to customize and extend
- **Community**: Large ecosystem and support

vs Kubernetes: Overkill for single-server homelab

vs Docker Swarm: Limited features, less active development

### Why Separate Stacks?

**Benefits:**
- Easier to manage specific service groups
- Better resource isolation
- Selective deployment (only what you need)
- Independent update cycles
- Clear responsibility boundaries

**Tradeoffs:**
- More compose files to manage
- Cross-stack communication requires planning
- Slight resource overhead (separate networks)

### Why LinuxServer.io Images?

- **Consistency**: PUID/PGID support across all images
- **Quality**: Well-maintained, regularly updated
- **Documentation**: Comprehensive guides
- **Community**: Active support and forums
- **Security**: Regular security patches

### Storage Path Choices

**`/srv/containers/`**: Standard for service data on Linux
- FHS (Filesystem Hierarchy Standard) compliant
- Clear purpose indication
- Easy to backup

**`/var/lib/containers/appdata/`**: Application state data
- Follows Docker conventions
- Separate from code/config
- Can be on different filesystem/disk

### Systemd Integration

**Why systemd?**
- Standard on modern Linux distributions
- Automatic service restart
- Boot-time startup
- Logging integration
- Service dependency management

**Alternative**: Cron, custom scripts (less robust)

## Scalability Considerations

### Current Architecture Limitations

- **Single server**: No built-in clustering
- **No load balancing**: Direct service access
- **Limited redundancy**: Single point of failure

### Scaling Options

1. **Vertical Scaling**
   - Add more CPU/RAM to host
   - Use faster storage (SSD/NVMe)
   - Optimize container resources

2. **Horizontal Scaling** (requires changes)
   - Multiple hosts with Docker Swarm/K8s
   - Load balancers (Traefik, Nginx)
   - Shared storage (NFS, Ceph)
   - Database clustering

3. **Service-Level Scaling**
   - Multiple instances of services
   - Queue-based processing
   - Caching layers (Redis, Memcached)
   - CDN for static content

## Monitoring and Observability

### Current Approach

- Docker logs: `docker compose logs`
- Resource monitoring: `docker stats`
- Service health: `docker compose ps`
- System logs: `journalctl`

### Future Enhancements

- Prometheus for metrics collection
- Grafana for visualization
- Loki for log aggregation
- Healthcheck endpoints
- Alerting systems

## Maintenance and Updates

### Update Strategy

1. **Pull new images**: `docker compose pull`
2. **Recreate containers**: `docker compose up -d`
3. **Cleanup old images**: `docker image prune`

### Backup Strategy

1. **Configuration backup**: Git repository
2. **Data backup**: `/var/lib/containers/appdata/`
3. **Database dumps**: Service-specific procedures
4. **Testing**: Regular restore validation

### Disaster Recovery

1. **Fresh system**: Install Docker
2. **Restore configs**: Clone git repository
3. **Restore data**: Extract backup archives
4. **Start services**: `docker compose up -d`
5. **Verify**: Check service accessibility

## Conclusion

This architecture provides a flexible, maintainable foundation for a homelab stack. It balances simplicity with functionality, making it accessible for home users while providing room for growth and customization.

For specific implementation details, refer to:
- [README.md](README.md) - Quick start guide
- [DEPLOYMENT.md](DEPLOYMENT.md) - Deployment procedures
- [FAQ.md](FAQ.md) - Common questions
- Stack-specific READMEs - Service details
