# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Comprehensive documentation suite
  - CONTRIBUTING.md for contributor guidelines
  - FAQ.md for frequently asked questions
  - ARCHITECTURE.md for system architecture overview
  - Enhanced README.md with better organization
  - Improved DEPLOYMENT.md with advanced topics
- Enhanced stack-specific READMEs
- Example systemd service files
- Troubleshooting guides

### Changed
- Improved documentation structure and organization
- Updated environment variable examples
- Enhanced security documentation

### Fixed
- Documentation formatting and typos
- Missing configuration examples

## [1.0.0] - Initial Release

### Added
- Initial Docker Compose stack structure
- Media stack configuration (placeholder)
- Web stack with:
  - Overseerr for media requests
  - Wizarr for user management
  - Organizr for unified dashboard
  - Homepage for application dashboard
- Cloud stack with:
  - Nextcloud for file storage and collaboration
  - Immich for photo/video backup
  - Collabora Online for document editing
- Basic documentation:
  - README.md with quick start guide
  - DEPLOYMENT.md with installation instructions
  - Stack-specific READMEs
- Environment variable examples (.env.example)
- Docker network configurations
- Systemd integration examples

### Infrastructure
- Git repository setup
- .gitignore configuration
- Directory structure organization

## Notes

### Version Numbering
- Major version (X.0.0): Breaking changes or major feature additions
- Minor version (0.X.0): New features, backward compatible
- Patch version (0.0.X): Bug fixes and minor improvements

### Upgrade Notes

When upgrading between versions:
1. Review the changelog for breaking changes
2. Backup your configuration and data
3. Update your `.env` files with any new variables
4. Pull new images: `docker compose pull`
5. Restart services: `docker compose up -d`

### Future Plans
- Additional media stack services (Radarr, Sonarr, Prowlarr, etc.)
- Enhanced monitoring and logging solutions
- Automated backup solutions
- Network infrastructure improvements (Traefik, VPN)
- Performance optimization guides
- Additional integration examples
