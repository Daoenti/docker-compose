# Security Best Practices

## Completed Security Measures

The following security measures have already been implemented:

✅ **Secret Management**
- `.env` files properly excluded from git via `.gitignore`
- `.env.example` templates created for all services
- No secrets committed to version control

✅ **Service Hardening**
- Vaultwarden signups disabled (`SIGNUPS_ALLOWED: false`)
- Health checks implemented across all services
- Logging configuration with rotation limits (prevents disk exhaustion)
- All services use `restart: unless-stopped` for automatic recovery

✅ **Infrastructure**
- Network isolation configured (minio, n8n have dedicated bridge networks)
- Deprecated features removed (`links:` directive)
- Images updated to current stable versions (nginx 1.27)
- Watchtower configured for automatic security updates (daily checks)

✅ **Documentation**
- Comprehensive backup strategy documented in `BACKUP.md`
- Docker conventions documented in `CONVENTIONS.md`
- Security best practices documented (this file)

## Environment Variables Setup

### Initial Setup
1. Copy each `.env.example` file to `.env` in the same directory:
   ```bash
   cd media-management && cp .env.example .env
   cd ../minio && cp .env.example .env
   cd ../n8n && cp .env.example .env
   cd ../public-services && cp .env.example .env
   cd ../infra-services && cp .env.example .env
   ```

2. Edit each `.env` file and replace placeholder values with actual credentials
3. Never commit `.env` files to git (already configured in `.gitignore`)

### Generating Secure Credentials

**Random passwords:**
```bash
# Generate a 32-character password
openssl rand -base64 32

# Generate alphanumeric password
openssl rand -hex 16
```

**For Vaultwarden admin token:**
```bash
# Generate and hash
echo -n "your_password" | argon2 "$(openssl rand -base64 32)" -e -id -k 19456 -t 2 -p 1
```

**For n8n encryption key:**
```bash
openssl rand -hex 32
```

## Critical Security Issues to Address

### 1. Container Privileges
The following containers run with elevated privileges:
- `prometheus-nvidiasmi` - runs privileged (required for NVIDIA GPU access)
- `homeassistant` - runs privileged (required for hardware access)
- `prometheus` - runs as root user (should be fixed)
- `grafana` - runs as root user (should be fixed)

**Action items:**
- [ ] Review if privileged mode is truly necessary
- [ ] Configure specific capabilities instead where possible
- [ ] Run grafana and prometheus as non-root users

### 2. Network Security
- [ ] Consider using Docker secrets instead of environment variables for production
- [ ] Review which services truly need `network_mode: host`
- [ ] Implement network policies if using Docker Swarm or Kubernetes

### 3. Backup Strategy
A comprehensive backup strategy is documented in `BACKUP.md`, including:
- Backup schedules and retention policies
- Multiple backup methods (rsync, Docker volumes, Duplicati)
- Complete disaster recovery procedures
- 3-2-1 backup rule implementation

**Quick reference - Manual backup:**
```bash
#!/bin/bash
BACKUP_DIR="/backup/docker-compose-$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"

# Backup configs
rsync -av /opt/docker/ "$BACKUP_DIR/docker/"

# Backup data
rsync -av /opt/data/ "$BACKUP_DIR/data/"

# Backup n8n database
docker exec n8n-postgres pg_dump -U admin n8n > "$BACKUP_DIR/n8n-db.sql"
```

See `BACKUP.md` for automated solutions and complete restore procedures.

## Ongoing Security Maintenance

### Regular Tasks
- [ ] Review watchtower logs for automatic updates
- [ ] Check for security vulnerabilities: `docker scan <image>`
- [ ] Rotate credentials quarterly
- [ ] Review container logs for suspicious activity
- [ ] Update base images regularly
- [ ] Monitor resource usage for anomalies

### Monitoring Recommendations
- Set up Grafana alerts for:
  - High CPU/memory usage
  - Failed authentication attempts
  - Disk space thresholds
  - Container restart loops

### Access Control
- Limit access to the Docker socket (`/var/run/docker.sock`)
- Use SSH key authentication for server access
- Implement fail2ban or similar for brute force protection
- Consider using a reverse proxy (Traefik/Nginx Proxy Manager) with:
  - SSL/TLS certificates
  - Rate limiting
  - Authentication middleware

## Emergency Response

### If Credentials are Compromised
1. Immediately rotate all affected credentials
2. Review container logs for unauthorized access
3. Check for unauthorized file modifications
4. Consider rebuilding affected containers
5. Update firewall rules if needed

### If System is Breached
1. Disconnect affected services from network
2. Capture container logs: `docker logs <container> > breach-log.txt`
3. Save container state: `docker commit <container> breach-investigation`
4. Review all access logs
5. Restore from clean backup after securing the system

## Additional Resources
- [Docker Security Best Practices](https://docs.docker.com/develop/security-best-practices/)
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
- [OWASP Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)
