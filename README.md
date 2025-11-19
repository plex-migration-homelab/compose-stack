# Docker Compose Homelab Stack

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Docker](https://img.shields.io/badge/docker-compose-blue.svg)](https://docs.docker.com/compose/)

A comprehensive, production-ready Docker Compose stack for running a complete homelab environment. This repository provides modular, well-documented configurations for media management, web applications, and cloud services.

## 📋 Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Quick Start](#quick-start)
- [Stack Components](#stack-components)
- [Documentation](#documentation)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [Contributing](#contributing)
- [Support](#support)
- [License](#license)

## 🌟 Overview

This repository contains Docker Compose configurations for a complete homelab stack organized into three main categories:

- **🎬 Media Stack** (`media/`) - Media management and streaming services
- **🌐 Web Stack** (`web/`) - Web applications and dashboards
- **☁️ Cloud Stack** (`cloud/`) - Self-hosted cloud services and collaboration tools

Each stack is independent, allowing you to deploy only the services you need.

## ✨ Features

- **🔧 Modular Design**: Deploy only what you need
- **📦 Easy Configuration**: Simple `.env` file configuration
- **🔄 Auto-restart**: Systemd integration for automatic startup
- **🔒 Secure by Default**: Isolated networks and proper permission handling
- **📚 Well Documented**: Comprehensive guides and examples
- **🚀 Production Ready**: Battle-tested configurations
- **🔁 Easy Updates**: Simple `docker compose pull` and restart
- **💾 Data Persistence**: Clear separation of config and data

## 🚀 Quick Start

```bash
# 1. Clone the repository
sudo git clone https://github.com/plex-migration-homelab/compose-stack.git /srv/containers
cd /srv/containers

# 2. Choose a stack and configure it
cd web
cp .env.example .env
nano .env  # Edit with your values

# 3. Create data directories
sudo mkdir -p /var/lib/containers/appdata
sudo chown -R $(id -u):$(id -g) /var/lib/containers/appdata

# 4. Deploy the stack
docker compose up -d

# 5. Check status
docker compose ps
```

## 📦 Stack Components

### 🎬 Media Stack

Comprehensive media management solution:

- **Plex/Jellyfin** - Media server for streaming
- **Radarr** - Movie collection management
- **Sonarr** - TV show collection management
- **Lidarr** - Music collection management
- **Prowlarr** - Indexer management
- **qBittorrent** - Download client

> **Note**: Media stack services are examples. Add the services you need to `media/compose.yml`.

[📖 Media Stack Documentation](media/README.md)

### 🌐 Web Stack

Web applications and dashboards:

- **Overseerr** (Port 5055) - Media request and discovery tool
- **Wizarr** (Port 5690) - User invitation and management
- **Organizr** (Port 9983) - Unified dashboard
- **Homepage** (Port 3000) - Modern application dashboard

[📖 Web Stack Documentation](web/README.md)

### ☁️ Cloud Stack

Self-hosted cloud services:

- **Nextcloud** (Ports 8080/8443) - Complete cloud solution
  - File storage and sharing
  - Calendar and contacts
  - Extensible app ecosystem
- **Immich** (Port 2283) - Photo and video backup (Google Photos alternative)
- **Collabora Online** (Port 9980) - Office suite for Nextcloud

[📖 Cloud Stack Documentation](cloud/README.md)

## 📚 Documentation

📋 **[DOCS_INDEX.md](DOCS_INDEX.md)** - Complete documentation index with search guide

### Core Documentation
- **[DEPLOYMENT.md](DEPLOYMENT.md)** - Complete deployment guide with systemd integration
- **[ARCHITECTURE.md](ARCHITECTURE.md)** - System architecture and design decisions
- **[QUICK_REFERENCE.md](QUICK_REFERENCE.md)** - Fast reference for common commands

### Guides & Help
- **[FAQ.md](FAQ.md)** - Frequently asked questions and troubleshooting
- **[SERVICE_TEMPLATE.md](SERVICE_TEMPLATE.md)** - Template for adding new services

### Project Information
- **[CONTRIBUTING.md](CONTRIBUTING.md)** - Guidelines for contributing
- **[SECURITY.md](SECURITY.md)** - Security policies and best practices
- **[CHANGELOG.md](CHANGELOG.md)** - Version history and updates
- **[LICENSE](LICENSE)** - MIT License

## 🔧 Prerequisites

- **Docker Engine** 24.0 or later
- **Docker Compose** v2.20 or later
- **Linux** system with systemd
- **Root/sudo access** for initial setup
- **Minimum 4GB RAM** (8GB+ recommended)
- **20GB+ disk space** (plus storage for your data)

### Install Docker

```bash
# Install Docker Engine
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add your user to docker group
sudo usermod -aG docker $USER

# Enable and start Docker
sudo systemctl enable docker
sudo systemctl start docker

# Verify installation
docker --version
docker compose version
```

## 💿 Installation

### 1. Clone Repository

```bash
# Create containers directory
sudo mkdir -p /srv/containers

# Clone repository
sudo git clone https://github.com/plex-migration-homelab/compose-stack.git /srv/containers
cd /srv/containers

# Set proper ownership
sudo chown -R $USER:$USER /srv/containers
```

### 2. Create Data Directories

```bash
# Create appdata directory
sudo mkdir -p /var/lib/containers/appdata

# Set ownership (use your PUID/PGID)
sudo chown -R $(id -u):$(id -g) /var/lib/containers/appdata

# Verify
ls -la /var/lib/containers/
```

## ⚙️ Configuration

### Environment Variables

Each stack requires configuration via `.env` file:

```bash
cd /srv/containers/[stack-name]
cp .env.example .env
nano .env
```

**Common variables:**

```bash
# User and Group IDs (run 'id -u' and 'id -g' to find yours)
PUID=1000
PGID=1000

# Timezone (e.g., America/New_York, Europe/London)
TZ=America/New_York
```

**Stack-specific variables:**

- **Cloud Stack**: Database passwords, Collabora credentials
- **Media Stack**: Media library paths
- **Web Stack**: (Optional port overrides)

⚠️ **Security Note**: Always change default passwords in `.env` files!

### Directory Structure

```
/srv/containers/              # Stack configurations
├── media/                    # Media management stack
├── web/                      # Web applications stack
├── cloud/                    # Cloud services stack
└── [documentation files]

/var/lib/containers/appdata/  # Application data
├── overseerr/               # Service data directories
├── nextcloud/
├── immich/
└── [other services]
```

## 🎯 Usage

### Starting Services

```bash
# Start a specific stack
cd /srv/containers/[stack-name]
docker compose up -d

# Start all stacks
cd /srv/containers
for stack in media web cloud; do
  cd $stack && docker compose up -d && cd ..
done
```

### Managing Services

```bash
# View running containers
docker compose ps

# View logs
docker compose logs -f [service-name]

# Restart a service
docker compose restart [service-name]

# Stop a stack
docker compose down

# Update containers
docker compose pull
docker compose up -d
```

### Accessing Services

Once deployed, access services at:

**Web Stack:**
- Overseerr: http://localhost:5055
- Wizarr: http://localhost:5690
- Organizr: http://localhost:9983
- Homepage: http://localhost:3000

**Cloud Stack:**
- Nextcloud: http://localhost:8080
- Immich: http://localhost:2283
- Collabora: (via Nextcloud integration)

Replace `localhost` with your server's IP for remote access.

### Systemd Integration (Automatic Startup)

Create systemd services for automatic startup:

```bash
# Example for web stack
sudo nano /etc/systemd/system/docker-compose-web.service
```

Add:

```ini
[Unit]
Description=Docker Compose Web Stack
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/srv/containers/web
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable docker-compose-web.service
sudo systemctl start docker-compose-web.service
```

Repeat for media and cloud stacks. See [DEPLOYMENT.md](DEPLOYMENT.md) for details.

## 🛠️ Maintenance

### Updating Containers

```bash
cd /srv/containers/[stack-name]
docker compose pull          # Pull latest images
docker compose up -d         # Recreate containers with new images
docker image prune -f        # Clean up old images
```

### Backup

**Configuration backup:**
```bash
cd /srv/containers
tar -czf ~/homelab-config-$(date +%F).tar.gz \
  --exclude='.git' --exclude='*.env' \
  media/ web/ cloud/ *.md
```

**Data backup:**
```bash
sudo tar -czf ~/homelab-appdata-$(date +%F).tar.gz \
  -C /var/lib/containers appdata/
```

For detailed backup and restore procedures, see [DEPLOYMENT.md](DEPLOYMENT.md).

### Monitoring

```bash
# Resource usage
docker stats

# Service status
docker compose ps

# Logs
docker compose logs -f [service-name]

# System logs (with systemd)
journalctl -u docker-compose-web.service -f
```

## 🤝 Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for:

- How to report issues
- How to submit pull requests
- Development guidelines
- Code of conduct

## 💬 Support

### Getting Help

- **📖 Documentation**: Check the docs in this repository first
- **❓ FAQ**: See [FAQ.md](FAQ.md) for common questions
- **🐛 Issues**: [Report bugs or request features](https://github.com/plex-migration-homelab/compose-stack/issues)
- **💡 Discussions**: [Ask questions and share ideas](https://github.com/plex-migration-homelab/compose-stack/discussions)

### Troubleshooting

**Services won't start:**
1. Check logs: `docker compose logs [service-name]`
2. Verify `.env` configuration
3. Check port conflicts: `sudo lsof -i :[port]`
4. Verify permissions on appdata directories

**Permission errors:**
```bash
sudo chown -R $(id -u):$(id -g) /var/lib/containers/appdata/[service]
```

See [FAQ.md](FAQ.md) for more troubleshooting tips.

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- [LinuxServer.io](https://www.linuxserver.io/) for excellent Docker images
- [Docker](https://www.docker.com/) for containerization platform
- Open source community for the amazing applications
- All contributors to this project

## ⭐ Star History

If you find this project useful, please consider giving it a star!

---

**Project Status**: Active Development  
**Last Updated**: 2024

For questions, issues, or contributions, please visit the [GitHub repository](https://github.com/plex-migration-homelab/compose-stack).
