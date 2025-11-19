# Security Policy

## Reporting a Vulnerability

We take the security of this project seriously. If you discover a security vulnerability, please follow these steps:

### Reporting Process

1. **Do NOT** open a public issue for security vulnerabilities
2. Email the maintainers directly at: [security contact - update with actual email]
3. Include detailed information:
   - Description of the vulnerability
   - Steps to reproduce
   - Potential impact
   - Suggested fix (if any)

### What to Expect

- **Acknowledgment**: Within 48 hours
- **Initial Assessment**: Within 7 days
- **Fix Timeline**: Varies based on severity
- **Disclosure**: After fix is released (coordinated disclosure)

## Security Best Practices

### For Users

When deploying this stack, follow these security guidelines:

#### 1. Environment Variables

```bash
# NEVER commit .env files to version control
# Always change default passwords
# Use strong, unique passwords

# Example of secure password generation
openssl rand -base64 32
```

#### 2. Network Security

- **Use a reverse proxy** (Traefik, Nginx, Caddy) for external access
- **Enable SSL/TLS** with valid certificates (Let's Encrypt)
- **Configure firewall rules** (UFW, iptables)
- **Consider VPN** for remote access (WireGuard, Tailscale)

```bash
# Example UFW configuration
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
sudo ufw enable
```

#### 3. Container Security

- **Keep containers updated** regularly
  ```bash
  docker compose pull
  docker compose up -d
  ```
- **Use official or trusted images** (LinuxServer.io, official repos)
- **Run containers as non-root** where possible (PUID/PGID)
- **Limit container capabilities** and resources

#### 4. Access Control

- **Use strong authentication** for all services
- **Enable 2FA/MFA** where available
- **Implement role-based access control** (RBAC)
- **Regular audit** of user access and permissions
- **Use API keys** instead of passwords for automation

#### 5. Data Protection

- **Encrypt sensitive data** at rest and in transit
- **Regular backups** stored securely off-site
- **Test restore procedures** periodically
- **Secure backup encryption** with strong keys

```bash
# Example encrypted backup
tar -czf - /var/lib/containers/appdata | \
  openssl enc -aes-256-cbc -e -out backup-encrypted.tar.gz.enc
```

#### 6. Monitoring and Logging

- **Enable logging** for all services
- **Monitor for suspicious activity**
- **Set up alerts** for security events
- **Regular log review**

```bash
# Check Docker logs
docker compose logs --since 24h | grep -i error

# Check systemd logs
journalctl -u docker.service -n 100 | grep -i error
```

### Network Isolation

Services are isolated by default in separate networks:

```yaml
# Each stack has its own network
media_network
web_network
cloud_network
```

For cross-stack communication, use:
- External networks (explicitly configured)
- Reverse proxy (recommended)
- Host networking (least secure - avoid if possible)

### Secrets Management

#### Don't:
- ❌ Commit passwords to git
- ❌ Use default passwords
- ❌ Share .env files
- ❌ Store credentials in compose.yml

#### Do:
- ✅ Use .env files (git-ignored)
- ✅ Generate strong passwords
- ✅ Rotate credentials regularly
- ✅ Use Docker secrets for production

```yaml
# Example Docker secret usage
secrets:
  db_password:
    file: ./secrets/db_password.txt

services:
  db:
    secrets:
      - db_password
```

### External Access Security

If exposing services to the internet:

#### 1. Use Reverse Proxy with SSL

```nginx
# Example Nginx configuration
server {
    listen 443 ssl http2;
    server_name service.yourdomain.com;
    
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    
    location / {
        proxy_pass http://localhost:5055;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

#### 2. Configure Fail2ban

```ini
# /etc/fail2ban/jail.local
[nginx-limit-req]
enabled = true
filter = nginx-limit-req
logpath = /var/log/nginx/error.log
maxretry = 5
bantime = 3600
```

#### 3. Use Rate Limiting

```nginx
# Nginx rate limiting
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

location /api/ {
    limit_req zone=api_limit burst=20 nodelay;
}
```

### Docker Daemon Security

Secure Docker daemon configuration:

```json
// /etc/docker/daemon.json
{
  "icc": false,
  "userland-proxy": false,
  "live-restore": true,
  "no-new-privileges": true,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

### Service-Specific Security

#### Nextcloud

- Enable encryption at rest
- Use HTTPS only
- Configure trusted domains
- Enable 2FA for all users
- Regular security scans

```bash
# Run Nextcloud security scan
docker exec -u www-data nextcloud php occ security:certificates
```

#### Immich

- Change default admin password immediately
- Restrict upload permissions
- Enable authentication for API
- Regular backup of photos

#### Overseerr/Wizarr

- Use strong passwords
- Limit user permissions
- Enable Plex/Jellyfin authentication
- Regular user audit

### Compliance and Regulations

If storing sensitive data:

- **GDPR** compliance (if in EU or serving EU users)
- **HIPAA** compliance (if storing health data)
- **Data retention** policies
- **Right to erasure** procedures
- **Data export** capabilities

### Security Checklist

Before going to production:

- [ ] All default passwords changed
- [ ] .env files configured and secured
- [ ] Firewall rules configured
- [ ] SSL/TLS certificates installed
- [ ] Reverse proxy configured
- [ ] Backup system tested
- [ ] Monitoring and alerting enabled
- [ ] Documentation reviewed
- [ ] Security scan completed
- [ ] Access controls verified
- [ ] 2FA enabled where possible
- [ ] Logs configured and rotating
- [ ] Update schedule planned

### Regular Security Maintenance

#### Weekly
- Review access logs
- Check for failed login attempts
- Verify backup completion

#### Monthly
- Update containers
- Review user access
- Check for security advisories
- Test backup restoration

#### Quarterly
- Full security audit
- Password rotation
- Review and update firewall rules
- Security training/awareness

### Security Tools

Recommended security tools:

```bash
# Check for container vulnerabilities
docker scan [image-name]

# Check for outdated packages
docker exec [container] apt list --upgradable

# Monitor file integrity
sudo apt install aide
sudo aide --init

# Network security scanner
sudo apt install nmap
nmap -sV localhost
```

### Incident Response

If you suspect a security breach:

1. **Immediately**:
   - Isolate affected services
   - Stop containers: `docker compose down`
   - Preserve logs: `docker compose logs > incident.log`

2. **Assess**:
   - Determine scope of breach
   - Identify affected data
   - Document timeline

3. **Remediate**:
   - Patch vulnerabilities
   - Change all passwords
   - Review access logs
   - Restore from clean backup if needed

4. **Report**:
   - Notify affected users
   - Report to maintainers
   - Document lessons learned

### Additional Resources

- [Docker Security Best Practices](https://docs.docker.com/engine/security/)
- [OWASP Docker Security](https://owasp.org/www-project-docker-security/)
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
- [Docker Bench Security](https://github.com/docker/docker-bench-security)

## Security Updates

Security updates will be published in:
- [CHANGELOG.md](CHANGELOG.md)
- GitHub Security Advisories
- Release notes

## Contact

For security concerns, contact: [Update with actual contact information]

---

**Last Updated**: 2024

Remember: Security is an ongoing process, not a one-time setup. Stay vigilant and keep your systems updated!
