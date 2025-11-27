# Backup Strategy

This document outlines backup strategies for your Docker infrastructure and data.

## What Needs to be Backed Up

### Critical (Cannot Recreate)
1. **Configuration Files** - `/opt/docker/*/config` and `/opt/docker/*/data`
2. **Database Data** - PostgreSQL (n8n), SQLite databases (various services)
3. **Media Metadata** - Plex libraries, Radarr/Sonarr databases
4. **User Data** - Vaultwarden passwords, Home Assistant automations
5. **Environment Files** - `.env` files (stored securely, not in git)

### Important (Difficult to Recreate)
1. **Media Files** - `/opt/data/media/*` (movies, TV shows, etc.)
2. **Download History** - `/opt/data/usenet/downloads`
3. **Application State** - Radarr/Sonarr/Prowlarr indexed data

### Recoverable (Can Recreate from Docker Compose)
1. **Container Images** - Can be pulled again
2. **Docker Networks** - Defined in compose files
3. **Base Configuration** - Defined in compose files

## Backup Schedules

### Recommended Schedule

| Data Type | Frequency | Retention | Method |
|-----------|-----------|-----------|--------|
| Databases | Daily | 30 days | Full backup |
| Configs | Daily | 30 days | Incremental |
| Media | Weekly | 4 weeks | Incremental |
| .env files | After changes | Forever | Encrypted |
| Compose files | On commit | Git history | Version control |

## Backup Methods

### Method 1: Simple Rsync Script

```bash
#!/bin/bash
# /usr/local/bin/docker-backup.sh

BACKUP_ROOT="/backup/docker"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="$BACKUP_ROOT/$DATE"

# Create backup directory
mkdir -p "$BACKUP_DIR"

echo "Starting backup at $(date)"

# Backup all docker configs
echo "Backing up configurations..."
rsync -av --exclude='*.log' --exclude='cache/*' \
  /opt/docker/ "$BACKUP_DIR/docker/"

# Backup media metadata (not the media itself - too large)
echo "Backing up metadata..."
rsync -av --exclude='*.mp4' --exclude='*.mkv' --exclude='*.avi' \
  /opt/data/ "$BACKUP_DIR/data/"

# Backup environment files (encrypted)
echo "Backing up environment files..."
tar -czf - /home/tyohn/github/docker-compose/*/.env | \
  gpg --encrypt --recipient your@email.com \
  > "$BACKUP_DIR/env-files.tar.gz.gpg"

# Backup databases
echo "Backing up databases..."
docker exec n8n-postgres pg_dump -U admin n8n | \
  gzip > "$BACKUP_DIR/n8n-db.sql.gz"

# Keep only last 30 days of backups
echo "Cleaning old backups..."
find "$BACKUP_ROOT" -type d -mtime +30 -exec rm -rf {} +

echo "Backup completed at $(date)"
echo "Backup size: $(du -sh $BACKUP_DIR)"
```

### Method 2: Docker Volume Backup

```bash
#!/bin/bash
# Backup specific Docker volumes

VOLUME_NAME="n8n_n8n_storage"
BACKUP_FILE="/backup/volumes/${VOLUME_NAME}_$(date +%Y%m%d).tar.gz"

docker run --rm \
  -v $VOLUME_NAME:/data \
  -v /backup/volumes:/backup \
  alpine tar czf /backup/$(basename $BACKUP_FILE) -C /data .
```

### Method 3: Using Duplicati (Recommended)

Duplicati provides:
- Automated scheduled backups
- Encryption
- Incremental backups
- Multiple destination support (local, cloud)
- Web interface

**Setup:**
```yaml
# Add to infra-services/docker-compose.yml
  duplicati:
    image: lscr.io/linuxserver/duplicati:latest
    container_name: duplicati
    environment:
      - PUID=1000
      - PGID=973
      - TZ=Etc/UTC
    volumes:
      - /opt/docker/duplicati/config:/config
      - /opt/docker:/source/docker:ro
      - /opt/data:/source/data:ro
      - /backup:/backups
    ports:
      - 8200:8200
    restart: unless-stopped
```

## Backup Destinations

### Local Options
- **NAS** - Network attached storage
- **External Drive** - USB hard drive (automated with scripts)
- **Secondary Server** - Rsync to another machine

### Cloud Options
- **Backblaze B2** - Affordable, S3-compatible
- **AWS S3** - Reliable, many regions available
- **Wasabi** - S3-compatible, no egress fees
- **Your MinIO Cluster** - Use the MinIO cluster you already have!

### 3-2-1 Rule

Follow the 3-2-1 backup strategy:
- **3** copies of data (original + 2 backups)
- **2** different media types (disk + cloud)
- **1** copy offsite

## Restore Procedures

### Restore Configuration

```bash
# Stop all services
cd /home/tyohn/github/docker-compose
for dir in */; do
  cd "$dir" && docker compose down && cd ..
done

# Restore from backup
rsync -av /backup/docker/20250101_120000/docker/ /opt/docker/

# Start services
for dir in */; do
  cd "$dir" && docker compose up -d && cd ..
done
```

### Restore Databases

```bash
# Restore n8n database
cat /backup/docker/20250101_120000/n8n-db.sql.gz | \
  gunzip | \
  docker exec -i n8n-postgres psql -U admin n8n
```

### Restore from Duplicati

1. Access Duplicati web interface (http://localhost:8200)
2. Select backup set
3. Choose restore destination
4. Select files/folders to restore
5. Run restore job

## Disaster Recovery Plan

### Complete System Failure

1. **Reinstall Host OS** (if necessary)
2. **Install Docker**
   ```bash
   curl -fsSL https://get.docker.com | sh
   usermod -aG docker $USER
   ```

3. **Restore Docker Compose Files**
   ```bash
   git clone <your-repo> /home/tyohn/github/docker-compose
   ```

4. **Restore Configurations**
   ```bash
   rsync -av /backup/docker/latest/ /opt/docker/
   ```

5. **Restore Environment Files**
   ```bash
   gpg --decrypt /backup/docker/latest/env-files.tar.gz.gpg | \
     tar -xzf - -C /
   ```

6. **Create Backend Network**
   ```bash
   docker network create backend
   ```

7. **Start All Services**
   ```bash
   cd /home/tyohn/github/docker-compose
   for dir in */; do
     cd "$dir" && docker compose up -d && cd ..
   done
   ```

8. **Restore Databases** (if needed)
9. **Verify All Services** - Check health status

### Single Service Failure

1. Stop the failed service
2. Restore configuration from backup
3. Restart the service
4. Verify functionality

### Data Corruption

1. Stop affected service
2. Restore data from last known good backup
3. Restore database if applicable
4. Restart service
5. Verify data integrity

## Monitoring Backups

### Check Backup Script Status

```bash
# View last backup log
tail -100 /var/log/docker-backup.log

# Check backup sizes
du -sh /backup/docker/*

# Count number of backups
ls -1 /backup/docker | wc -l
```

### Automated Monitoring

Add to your backup script:
```bash
# Send notification on failure
if [ $? -ne 0 ]; then
  echo "Backup failed!" | mail -s "Docker Backup Failed" admin@example.com
fi

# Log to syslog
logger -t docker-backup "Backup completed successfully"
```

### Grafana Dashboard

Monitor backup metrics:
- Last backup timestamp
- Backup size trends
- Failed backup count
- Backup duration

## Testing Backups

**Critical**: Regularly test restores to ensure backups work!

### Monthly Test Checklist

- [ ] Restore a configuration directory to test location
- [ ] Restore a database dump
- [ ] Verify restored data is complete and uncorrupted
- [ ] Test a single service restoration
- [ ] Document any issues found

### Annual Test

- [ ] Full disaster recovery simulation
- [ ] Restore complete system from backup
- [ ] Verify all services work correctly
- [ ] Update disaster recovery documentation

## Automation with Cron

```bash
# Edit crontab
crontab -e

# Add backup jobs
# Daily config backup at 2 AM
0 2 * * * /usr/local/bin/docker-backup.sh >> /var/log/docker-backup.log 2>&1

# Weekly full backup at 3 AM Sunday
0 3 * * 0 /usr/local/bin/docker-backup-full.sh >> /var/log/docker-backup.log 2>&1

# Daily database backup at 1 AM
0 1 * * * docker exec n8n-postgres pg_dump -U admin n8n | gzip > /backup/n8n-$(date +\%Y\%m\%d).sql.gz
```

## Security Considerations

1. **Encrypt Backups** - Especially for cloud storage
2. **Secure .env Files** - These contain passwords and API keys
3. **Restrict Access** - Limit who can access backups
4. **Audit Logs** - Track backup access and changes
5. **Test Restores** - In isolated environment first

## Cost Estimates

### Storage Requirements

Approximate sizes:
- Docker configs: ~5-10 GB
- Databases: ~1-2 GB
- Media metadata: ~5-10 GB
- **Total per backup**: ~15-25 GB

With 30 days retention and daily backups:
- **Total storage needed**: ~500 GB - 750 GB

### Cloud Backup Costs (Monthly)

- **Backblaze B2**: $0.005/GB = ~$2.50-3.75/month
- **AWS S3** (Standard): $0.023/GB = ~$11.50-17.25/month
- **Wasabi**: $5.99/TB minimum = $5.99/month

## Additional Resources

- [Docker Backup and Restore](https://docs.docker.com/storage/volumes/#backup-restore-or-migrate-data-volumes)
- [Duplicati Documentation](https://duplicati.readthedocs.io/)
- [Rsync Tutorial](https://www.digitalocean.com/community/tutorials/how-to-use-rsync-to-sync-local-and-remote-directories)
