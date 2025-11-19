# Service Addition Template

Use this template when adding new services to your homelab stack.

## Pre-Addition Checklist

- [ ] Service is needed and will be actively used
- [ ] Service has a reputable Docker image available
- [ ] Resource requirements are understood
- [ ] Integration needs are identified
- [ ] Security implications are considered

## Service Information

### Basic Details

**Service Name**: [e.g., Radarr]  
**Purpose**: [Brief description of what the service does]  
**Image Source**: [e.g., lscr.io/linuxserver/radarr:latest]  
**Official Documentation**: [Link to official docs]  
**Category**: [Media/Web/Cloud/Other]

### Resource Requirements

**Minimum:**
- CPU: [e.g., 1 core]
- RAM: [e.g., 512MB]
- Disk: [e.g., 1GB for config]

**Recommended:**
- CPU: [e.g., 2 cores]
- RAM: [e.g., 2GB]
- Disk: [e.g., 5GB for config]

### Network Requirements

**Ports Needed:**
- Port: [e.g., 7878] - Purpose: [e.g., Web UI]
- Port: [e.g., 9898] - Purpose: [e.g., API]

**DNS/Hostname**: [if applicable]

### Dependencies

**Required Services:**
- [ ] Database: [type/version if needed]
- [ ] Cache: [Redis/Memcached if needed]
- [ ] Other: [any other services]

**Integration With:**
- [ ] Service 1: [how they integrate]
- [ ] Service 2: [how they integrate]

## Docker Compose Configuration

### Basic Service Definition

```yaml
services:
  service-name:
    image: namespace/image:tag
    container_name: service-name
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      # Add service-specific environment variables
    volumes:
      - /var/lib/containers/appdata/service-name:/config
      # Add additional volumes
    ports:
      - "PORT:PORT"
    restart: unless-stopped
    # Add depends_on if needed
    # depends_on:
    #   - database
```

### With Database

If service requires a database:

```yaml
services:
  service-name:
    image: namespace/image:tag
    container_name: service-name
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - DB_HOST=service-db
      - DB_NAME=service
      - DB_USER=service
      - DB_PASSWORD=${SERVICE_DB_PASSWORD}
    volumes:
      - /var/lib/containers/appdata/service-name:/config
    ports:
      - "PORT:PORT"
    restart: unless-stopped
    depends_on:
      - service-db

  service-db:
    image: postgres:15  # or mariadb:10.11
    container_name: service-db
    environment:
      - POSTGRES_DB=service
      - POSTGRES_USER=service
      - POSTGRES_PASSWORD=${SERVICE_DB_PASSWORD}
    volumes:
      - /var/lib/containers/appdata/service-name/db:/var/lib/postgresql/data
    restart: unless-stopped
```

### With Resource Limits

```yaml
services:
  service-name:
    image: namespace/image:tag
    container_name: service-name
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - /var/lib/containers/appdata/service-name:/config
    ports:
      - "PORT:PORT"
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          memory: 512M
```

### With Healthcheck

```yaml
services:
  service-name:
    image: namespace/image:tag
    container_name: service-name
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - /var/lib/containers/appdata/service-name:/config
    ports:
      - "PORT:PORT"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:PORT/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

## Environment Variables

Add to `.env.example`:

```bash
# Service Name Configuration
SERVICE_PORT=PORT_NUMBER
SERVICE_DB_PASSWORD=changeme_secure_password
SERVICE_API_KEY=your_api_key_here
# Add other service-specific variables
```

## Data Directories

Create required directories:

```bash
# Create service data directory
sudo mkdir -p /var/lib/containers/appdata/service-name

# Create subdirectories if needed
sudo mkdir -p /var/lib/containers/appdata/service-name/config
sudo mkdir -p /var/lib/containers/appdata/service-name/data
sudo mkdir -p /var/lib/containers/appdata/service-name/db

# Set ownership
sudo chown -R $(id -u):$(id -g) /var/lib/containers/appdata/service-name
```

## Security Considerations

### Required Actions

- [ ] Change default credentials
- [ ] Configure authentication
- [ ] Set up access controls
- [ ] Review exposed ports
- [ ] Configure firewall rules
- [ ] Enable SSL/TLS if applicable
- [ ] Review permissions

### Security Notes

[Document any security-specific configuration needed]

## Initial Setup Steps

### 1. Add to compose.yml

```bash
cd /srv/containers/[appropriate-stack]
nano compose.yml
# Add service configuration
```

### 2. Update Environment File

```bash
cp .env.example .env.example.new
# Add new variables
nano .env
# Configure service-specific variables
```

### 3. Create Data Directories

```bash
sudo mkdir -p /var/lib/containers/appdata/service-name
sudo chown -R $(id -u):$(id -g) /var/lib/containers/appdata/service-name
```

### 4. Deploy Service

```bash
docker compose up -d service-name
```

### 5. Verify Deployment

```bash
# Check status
docker compose ps service-name

# Check logs
docker compose logs -f service-name

# Test access
curl http://localhost:PORT
```

### 6. Initial Configuration

1. Access the web UI at `http://localhost:PORT`
2. Complete initial setup wizard
3. Configure authentication
4. Set up integrations
5. Test functionality

## Integration Setup

### With Other Services

Document how to integrate with existing services:

**Example: Integrating with Plex**
1. Get Plex token: [instructions]
2. In service settings, add Plex URL and token
3. Test connection
4. Configure sync settings

**Example: Integrating with Radarr/Sonarr**
1. Get API key from Radarr/Sonarr
2. In service settings, add URL and API key
3. Configure quality profiles
4. Test connection

## Documentation Updates

### Update Stack README

Add service to stack README.md:

```markdown
### Service Name
**Port:** PORT_NUMBER
Brief description of the service and its purpose.

**Access:** `http://localhost:PORT`

**Initial Setup:**
1. Step one
2. Step two
3. Step three
```

### Update Main README

If significant, add to main README.md stack components section.

## Testing Checklist

- [ ] Service starts successfully
- [ ] Web UI is accessible
- [ ] Authentication works
- [ ] Integrations function correctly
- [ ] Data persists across restarts
- [ ] Logs show no errors
- [ ] Resource usage is acceptable
- [ ] Backups include new service

## Backup Configuration

Add to backup script:

```bash
# Backup service data
sudo tar -czf ~/service-name-backup-$(date +%F).tar.gz \
  -C /var/lib/containers/appdata service-name/
```

For database services:

```bash
# Database backup
docker exec service-db mysqldump -u service -p service > service-backup.sql
# or for PostgreSQL
docker exec service-db pg_dump -U service service > service-backup.sql
```

## Monitoring Setup

### Logs

```bash
# View service logs
docker compose logs -f service-name

# Save logs to file
docker compose logs service-name > service-logs.txt
```

### Resource Monitoring

```bash
# Monitor service resources
docker stats service-name
```

### Health Checks

```bash
# Check service health
docker inspect --format='{{.State.Health.Status}}' service-name
```

## Troubleshooting Guide

### Common Issues

**Issue: Service won't start**
```bash
# Check logs
docker compose logs service-name

# Check permissions
ls -la /var/lib/containers/appdata/service-name

# Recreate container
docker compose up -d --force-recreate service-name
```

**Issue: Can't access web UI**
```bash
# Check if service is running
docker compose ps service-name

# Check port binding
sudo netstat -tulpn | grep PORT

# Check firewall
sudo ufw status
```

**Issue: Integration not working**
- Verify API keys are correct
- Check network connectivity between containers
- Review service logs for errors
- Verify URLs are correct

## Removal Procedure

If you need to remove the service:

```bash
# Stop and remove container
docker compose down service-name

# Remove data (CAUTION: PERMANENT)
sudo rm -rf /var/lib/containers/appdata/service-name

# Remove from compose.yml
nano compose.yml
# Delete service definition

# Remove from .env
nano .env
# Delete service-specific variables
```

## Additional Resources

### Official Resources
- Official Documentation: [link]
- GitHub Repository: [link]
- Discord/Forum: [link]

### Community Resources
- Community Guides: [links]
- Video Tutorials: [links]
- Example Configurations: [links]

## Notes

[Add any additional notes, tips, or considerations specific to this service]

---

## Example: Complete Radarr Addition

### Service Details
**Service Name**: Radarr  
**Purpose**: Movie collection manager for Usenet and BitTorrent  
**Image**: lscr.io/linuxserver/radarr:latest  
**Port**: 7878

### Complete Configuration

```yaml
services:
  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - /var/lib/containers/appdata/radarr:/config
      - ${MOVIES_PATH}:/movies
      - ${DOWNLOADS_PATH}:/downloads
    ports:
      - "7878:7878"
    restart: unless-stopped
```

### Environment Variables

```bash
# Radarr Configuration
MOVIES_PATH=/path/to/movies
DOWNLOADS_PATH=/path/to/downloads
```

### Deployment

```bash
# Add to media stack
cd /srv/containers/media
# Add configuration to compose.yml
docker compose up -d radarr

# Access at http://localhost:7878
```

---

**Template Version**: 1.0  
**Last Updated**: 2024

Use this template as a guide for adding new services to ensure consistency and completeness.
