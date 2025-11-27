# Docker Compose Home Server Infrastructure

A comprehensive, production-ready Docker Compose setup for running a complete home server infrastructure with media management, monitoring, automation, and public services.

## Overview

This repository contains 7 Docker Compose stacks managing 35+ services organized by function:

- **Infrastructure Services** - Core monitoring, DNS, and system utilities
- **Media Management** - Automated media acquisition and organization
- **Public Services** - User-facing applications and services
- **Monitoring** - Metrics collection and visualization
- **Storage** - Distributed object storage cluster
- **Automation** - Workflow automation and orchestration
- **Home Automation** - Smart home control and monitoring

## Architecture

### Network Topology

```
┌─────────────────────────────────────────────────────────────┐
│                      Host Network                           │
│  - AdGuard Home (DNS/DHCP)                                  │
│  - Plex (Media Server)                                      │
│  - Home Assistant (Device Discovery)                        │
└─────────────────────────────────────────────────────────────┘

┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
│ infra-services  │   │media-management │   │ public-services │
│   Network       │   │   Network       │   │   Network       │
├─────────────────┤   ├─────────────────┤   ├─────────────────┤
│ - Prometheus    │   │ - Radarr        │   │ - Foundry VTT   │
│ - IT Tools      │   │ - Sonarr        │   │ - Kavita        │
│ - Watchtower    │   │ - Prowlarr      │   │ - Vaultwarden   │
│ - Flaresolverr  │   │ - SABnzbd       │   │ - Pastebin      │
│ - NvidiaSMI     │   │ - Notifiarr     │   │ - Plex          │
└────────┬────────┘   └────────┬────────┘   └────────┬────────┘
         │                     │                     │
         │    ┌────────────────┴────────────────┐    │
         └────┤      Backend Network            ├────┘
              │         (external)              │
              └─────────────────────────────────┘

┌─────────────────┐   ┌─────────────────┐
│  Grafana Stack  │   │      MinIO      │
├─────────────────┤   ├─────────────────┤
│ - Grafana       │   │ - MinIO x4      │
│ - cAdvisor      │   │ - Nginx LB      │
└─────────────────┘   └─────────────────┘

┌─────────────────┐   ┌─────────────────┐
│      n8n        │   │  Home Assistant │
├─────────────────┤   ├─────────────────┤
│ - n8n App       │   │ - HASS          │
│ - PostgreSQL    │   └─────────────────┘
└─────────────────┘
```

## Service Stacks

### 1. Infrastructure Services (`infra-services/`)

Core infrastructure and monitoring services.

**Services:**
- **IT Tools** (`:8180`) - Collection of useful IT utilities and tools
- **Prometheus** (`:9090`) - Metrics collection and time-series database
- **Prometheus NVIDIA SMI** (`:9202`) - GPU metrics exporter
- **Watchtower** - Automatic container updates (checks daily)
- **Flaresolverr** (`:8191`) - Cloudflare bypass for web scraping
- **AdGuard Home** (`:80`, `:53`, `:3000`) - DNS-level ad blocking and DHCP server

**Key Features:**
- Network mode: `infra-services` + `backend`
- AdGuard uses host network for DNS/DHCP
- NVIDIA GPU monitoring for ML/transcoding workloads
- Automated container updates via Watchtower

### 2. Media Management (`media-management/`)

Automated media acquisition, organization, and management.

**Services:**
- **Radarr** (`:7878`) - Movie management and automation
- **Sonarr** (`:8989`) - TV show management and automation
- **Prowlarr** (`:9696`) - Indexer management for Radarr/Sonarr
- **SABnzbd** (`:8085`) - Usenet downloader
- **Overseerr** (`:5055`) - Media request management
- **Tdarr** (`:8265`, `:8266`) - Media transcoding and health checks
- **Bazarr** (`:6767`) - Subtitle management
- **Recyclarr** - Automated quality profile management
- **Watchstate** (`:8080`) - Multi-platform watch state sync
- **Notifiarr** (`:5454`) - Notification aggregation and integration

**Key Features:**
- Network mode: `media-management` + `backend`
- Automated downloads via Usenet
- Quality profiles managed by Recyclarr
- Watch state syncing across Plex/Jellyfin/Emby
- Discord/Slack notifications via Notifiarr

### 3. Public Services (`public-services/`)

User-facing applications and services.

**Services:**
- **Foundry VTT** (`:30000`) - Virtual tabletop for RPGs
- **Kavita** (`:5000`) - Comic/manga/ebook server
- **Vaultwarden** (`:8085`) - Self-hosted Bitwarden password manager
- **Pastebin** (`:6163`) - Minimalist pastebin service
- **Plex** (`:32400`) - Media streaming server with GPU transcoding

**Key Features:**
- Network mode: `public-services` + `backend`
- Plex uses host network for better discovery
- Vaultwarden signups disabled for security
- NVIDIA GPU acceleration for Plex transcoding
- 8GB RAM tmpfs for Plex transcode cache

### 4. Monitoring Stack (`grafana/`)

Metrics visualization and container monitoring.

**Services:**
- **Grafana** (`:3456`) - Metrics visualization and dashboards
- **cAdvisor** (`:8889`) - Container resource usage monitoring

**Key Features:**
- Integrates with Prometheus from infra-services
- Real-time container resource monitoring
- Pre-configured dashboards for system metrics

### 5. Object Storage (`minio/`)

Distributed, S3-compatible object storage cluster.

**Services:**
- **MinIO Node 1-4** - 4-node distributed storage cluster
- **Nginx** (`:9000`, `:9001`) - Load balancer for MinIO API and Console

**Key Features:**
- 4-node distributed erasure coded cluster
- S3-compatible API
- High availability with automated failover
- 8 data volumes (2 per node) for redundancy
- Web console for management

### 6. Workflow Automation (`n8n/`)

Self-hosted workflow automation platform.

**Services:**
- **n8n** (`:5678`) - Workflow automation and integration platform
- **PostgreSQL** - Database backend for n8n

**Key Features:**
- Visual workflow builder
- 200+ integrations
- PostgreSQL for persistent storage
- Encryption key support for sensitive data

### 7. Home Automation (`homeassistant/`)

Smart home control and automation hub.

**Services:**
- **Home Assistant** (`:8123`) - Smart home automation platform

**Key Features:**
- Host network mode for device discovery
- Privileged mode for hardware access
- Persistent configuration storage
- Integration with IoT devices

## Prerequisites

### System Requirements

- **OS:** Linux (tested on Arch Linux)
- **Docker:** 20.10+ with Docker Compose v2
- **NVIDIA GPU:** Optional, required for Plex transcoding and GPU monitoring
- **Storage:**
  - 100GB+ for Docker volumes and configs
  - Adequate storage for media library (varies)
- **Memory:** 16GB+ recommended
- **Network:** Static IP or DHCP reservation recommended

### Required Setup

1. **Install Docker and Docker Compose:**
   ```bash
   curl -fsSL https://get.docker.com | sh
   usermod -aG docker $USER
   ```

2. **Install NVIDIA Container Toolkit** (if using GPU):
   ```bash
   # Arch Linux
   sudo pacman -S nvidia-container-toolkit

   # Configure Docker
   sudo nvidia-ctk runtime configure --runtime=docker
   sudo systemctl restart docker
   ```

3. **Create directory structure:**
   ```bash
   sudo mkdir -p /opt/docker
   sudo mkdir -p /opt/data/media/{movies,tv,music}
   sudo mkdir -p /opt/data/usenet/downloads
   sudo chown -R $USER:docker /opt/docker /opt/data
   ```

4. **Create backend network:**
   ```bash
   docker network create backend
   ```

## Quick Start

### 1. Clone Repository

```bash
git clone <repository-url> ~/github/docker-compose
cd ~/github/docker-compose
```

### 2. Configure Environment Variables

Copy and edit `.env` files for each stack:

```bash
# Copy all .env.example files to .env
find . -name ".env.example" -execdir cp {} .env \;

# Edit each .env file with your credentials
# See SECURITY.md for credential generation instructions
```

**Required `.env` files:**
- `media-management/.env` - Notifiarr API keys and service URLs
- `minio/.env` - MinIO root credentials
- `n8n/.env` - Database and encryption credentials
- `public-services/.env` - Foundry, Vaultwarden, vault credentials
- `infra-services/.env` - Optional: Flaresolverr configuration

### 3. Start Services

**Start all stacks:**
```bash
for dir in */; do
  if [ -f "$dir/docker-compose.yml" ]; then
    cd "$dir"
    docker compose up -d
    cd ..
  fi
done
```

**Start individual stack:**
```bash
cd infra-services
docker compose up -d
```

**Check service health:**
```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
```

### 4. Initial Configuration

After starting services, complete initial setup:

1. **AdGuard Home:** http://localhost:80 - Set up DNS and DHCP
2. **Grafana:** http://localhost:3456 - Default login: admin/admin
3. **Prowlarr:** http://localhost:9696 - Add indexers
4. **Radarr/Sonarr:** Add quality profiles and root folders
5. **Vaultwarden:** http://localhost:8085 - Create admin account, then disable signups
6. **MinIO:** http://localhost:9001 - Login with MINIO_USER/MINIO_PASS
7. **n8n:** http://localhost:5678 - Set up admin account

## Service Access

### Web Interfaces

| Service | Port | URL | Purpose |
|---------|------|-----|---------|
| AdGuard Home | 80 | http://localhost:80 | DNS/Ad Blocking |
| IT Tools | 8180 | http://localhost:8180 | Utilities |
| Prometheus | 9090 | http://localhost:9090 | Metrics |
| Grafana | 3456 | http://localhost:3456 | Dashboards |
| Radarr | 7878 | http://localhost:7878 | Movies |
| Sonarr | 8989 | http://localhost:8989 | TV Shows |
| Prowlarr | 9696 | http://localhost:9696 | Indexers |
| SABnzbd | 8085 | http://localhost:8085 | Downloads |
| Overseerr | 5055 | http://localhost:5055 | Requests |
| Tdarr | 8265 | http://localhost:8265 | Transcoding |
| Bazarr | 6767 | http://localhost:6767 | Subtitles |
| Watchstate | 8080 | http://localhost:8080 | Watch Sync |
| Notifiarr | 5454 | http://localhost:5454 | Notifications |
| Plex | 32400 | http://localhost:32400/web | Media Server |
| Foundry VTT | 30000 | http://localhost:30000 | Tabletop RPG |
| Kavita | 5000 | http://localhost:5000 | Comics/Books |
| Vaultwarden | 8085 | http://localhost:8085 | Passwords |
| Pastebin | 6163 | http://localhost:6163 | Pastebin |
| MinIO Console | 9001 | http://localhost:9001 | Storage |
| MinIO API | 9000 | http://localhost:9000 | S3 API |
| n8n | 5678 | http://localhost:5678 | Automation |
| Home Assistant | 8123 | http://localhost:8123 | Smart Home |
| cAdvisor | 8889 | http://localhost:8889 | Metrics |
| Flaresolverr | 8191 | http://localhost:8191 | Proxy |

### Accessing from Other Devices

To access services from other devices on your network, replace `localhost` with your server's IP address.

For external access, consider setting up:
- Reverse proxy (Traefik, Nginx Proxy Manager)
- VPN (Wireguard, Tailscale)
- Cloudflare Tunnel

## Maintenance

### Update Containers

Watchtower automatically updates containers daily. To update manually:

```bash
cd <service-directory>
docker compose pull
docker compose up -d
```

### View Logs

```bash
# All services in a stack
cd <service-directory>
docker compose logs -f

# Specific service
docker compose logs -f <service-name>

# Last 100 lines
docker compose logs -f --tail=100 <service-name>
```

### Restart Services

```bash
# Restart entire stack
cd <service-directory>
docker compose restart

# Restart specific service
docker compose restart <service-name>
```

### Check Resource Usage

```bash
# Container stats
docker stats

# Disk usage
docker system df

# Container sizes
docker ps --size
```

### Backup

See `BACKUP.md` for comprehensive backup strategies including:
- Automated backup scripts
- Duplicati configuration
- Disaster recovery procedures
- 3-2-1 backup rule implementation

## Configuration Files

### Data Locations

**Application configs and data:**
```
/opt/docker/
├── radarr/
├── sonarr/
├── plex/
├── vaultwarden/
├── grafana/
└── ...
```

**Media and shared data:**
```
/opt/data/
├── media/
│   ├── movies/
│   ├── tv/
│   └── music/
└── usenet/
    └── downloads/
```

### Environment Variables

Each stack with sensitive configuration has:
- `.env.example` - Template showing required variables
- `.env` - Actual credentials (gitignored, not in repo)

## Security

This setup implements several security best practices:

✅ Secrets managed via `.env` files (never committed to git)
✅ Health checks on all services for automatic recovery
✅ Logging limits to prevent disk exhaustion
✅ Automatic security updates via Watchtower
✅ Network isolation between service stacks
✅ Vaultwarden signups disabled
✅ Non-root users where possible

**See `SECURITY.md` for:**
- Credential generation and rotation
- Container privilege review
- Emergency response procedures
- Security best practices

## Troubleshooting

### Common Issues

**Container won't start:**
```bash
# Check logs
docker compose logs <service-name>

# Check health status
docker inspect <container> --format='{{json .State.Health}}'
```

**Permission issues:**
```bash
# Fix ownership (adjust PUID/PGID as needed)
sudo chown -R 1000:973 /opt/docker/<service>
```

**Network issues:**
```bash
# Verify backend network exists
docker network ls | grep backend

# Recreate if missing
docker network create backend
```

**Out of disk space:**
```bash
# Clean up old images and containers
docker system prune -a

# Check log sizes
docker ps -q | xargs docker inspect --format='{{.Name}} {{.LogPath}}' | \
  while read name path; do echo "$name: $(du -h "$path" 2>/dev/null | cut -f1)"; done
```

### Health Check Failures

Most services have health checks. View health status:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
```

If a service is unhealthy, check logs and verify:
- Required ports are not in use
- Configuration files exist
- Environment variables are set correctly
- Dependencies are running (e.g., database for n8n)

## Documentation

- **`CONVENTIONS.md`** - Docker Compose patterns and conventions used across all stacks
- **`SECURITY.md`** - Security best practices, credential management, and emergency procedures
- **`BACKUP.md`** - Comprehensive backup and disaster recovery strategies

## Technology Stack

- **Container Runtime:** Docker 20.10+ with Docker Compose v2
- **Orchestration:** Docker Compose (stack-based architecture)
- **Monitoring:** Prometheus + Grafana + cAdvisor
- **Storage:** MinIO (S3-compatible distributed object storage)
- **Automation:** n8n (workflow automation)
- **GPU:** NVIDIA Container Toolkit for transcoding and ML workloads
- **Networking:** Docker bridge networks with external backend network
- **Updates:** Watchtower for automated container updates

## Contributing

When modifying this setup:

1. Follow conventions documented in `CONVENTIONS.md`
2. Update `.env.example` if adding new secrets
3. Add health checks to new services
4. Include logging configuration
5. Update this README with new services
6. Test changes before committing

## License

This is a personal infrastructure setup. Adapt as needed for your own use.

## Resources

- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Specification](https://docs.docker.com/compose/compose-file/)
- [LinuxServer.io Images](https://www.linuxserver.io/) - Many services use LSIO images
- [TRaSH Guides](https://trash-guides.info/) - Media server setup guides