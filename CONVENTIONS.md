# Docker Compose Conventions

This document explains the patterns and conventions used across all docker-compose files in this repository.

## Common Variables Pattern

Most compose files define common environment variables using YAML anchors:

```yaml
x-common-variables: &common-variables
  PUID: 1000
  PGID: 973
  TZ: Etc/UTC
```

These are then referenced in services using:
```yaml
environment:
  <<: *common-variables
```

### Why Separate Anchors Per File?

Each compose file maintains its own `x-common-variables` anchor rather than using a shared file because:

1. **Independence** - Each stack can be deployed separately without dependencies
2. **Customization** - Different stacks may need different common variables
3. **Clarity** - All configuration for a stack is in one file

### Current Common Variables

- `PUID: 1000` - User ID for file permissions
- `PGID: 973` - Group ID for file permissions (docker group)
- `TZ: Etc/UTC` - Timezone setting

To change these globally, update the anchor in each compose file.

## Environment Variable Patterns

### Default Values Syntax

Use `${VAR:-default}` for optional variables:

```yaml
environment:
  - LOG_LEVEL=${LOG_LEVEL:-info}
  - ENABLE_FEATURE=${ENABLE_FEATURE:-false}
```

This provides a default value if the variable is not set in `.env`.

### Required Variables Syntax

Use `${VAR}` for required variables:

```yaml
environment:
  - API_KEY=${API_KEY}
  - DATABASE_URL=${DATABASE_URL}
```

These will cause an error if not defined in `.env`.

### Current Usage

- **infra-services** - Uses defaults for flaresolverr (LOG_LEVEL, LOG_HTML, CAPTCHA_SOLVER)
- **Other services** - Use required syntax, rely on `.env` files

## Network Patterns

### Network Types

1. **Service-specific networks** - Isolate related services
   ```yaml
   networks:
     - media-management
   ```

2. **Shared backend network** - For cross-stack communication
   ```yaml
   networks:
     - backend
   ```
   The `backend` network is marked as `external: true` and must be created manually:
   ```bash
   docker network create backend
   ```

3. **Host network mode** - For services needing direct host access
   ```yaml
   network_mode: host
   ```
   Used by: AdGuard (DNS/DHCP), Plex (discovery), Home Assistant (hardware)

### Network Architecture

```
┌─────────────────┐
│ infra-services  │──┐
│   - prometheus  │  │
│   - grafana     │  │
└─────────────────┘  │
                     │
┌─────────────────┐  │    ┌─────────┐
│media-management │  ├────┤ backend │ (external)
│   - radarr      │  │    └─────────┘
│   - sonarr      │  │
└─────────────────┘  │
                     │
┌─────────────────┐  │
│ public-services │──┘
│   - plex        │
│   - vault       │
└─────────────────┘

┌─────────────────┐
│     minio       │ (isolated)
└─────────────────┘

┌─────────────────┐
│      n8n        │ (isolated)
└─────────────────┘
```

## Volume Patterns

All services use bind mounts to host directories:

- **Configuration**: `/opt/docker/<service>/config` or `/opt/docker/<service>/data`
- **Media/Data**: `/opt/data/<type>/<content>`

### Standard Paths

```
/opt/docker/          # Service configurations and app data
├── radarr/
├── sonarr/
├── plex/
└── ...

/opt/data/            # Media and shared data
├── media/
│   ├── movies/
│   ├── tv/
│   └── ...
└── usenet/
    └── downloads/
```

## Resource Management

### Logging Configuration

All services use JSON file logging with rotation:
```yaml
logging:
  driver: "json-file"
  options:
    max-size: "10m"
    max-file: "3"
```

This limits each container to 30MB of logs (3 files × 10MB).

### GPU Access

Services needing GPU access use the nvidia runtime:

```yaml
runtime: nvidia
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          count: all  # or specific count
          capabilities: [gpu]
```

Used by: Plex (transcoding), Stash (processing)

## Health Checks

Most services include health checks for better orchestration:

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://127.0.0.1:8080"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 60s
```

**Note**: `127.0.0.1` is used instead of `localhost` to avoid IPv6 issues.

### Services Without Health Checks

- **notifiarr** - Distroless container, no shell/utilities
- **pastebin** - Distroless container, no shell/utilities
- **watchtower** - Runs periodically, doesn't need health check

## Security Patterns

### Secret Management

- All secrets stored in `.env` files (gitignored)
- Each service directory has `.env.example` showing required variables
- Never commit actual `.env` files to git

### User/Group IDs

Most services run as `PUID=1000` and `PGID=973`:
- `1000` - Primary user on most systems
- `973` - Docker group (allows socket access where needed)

### Privileged Containers

Only used when necessary:
- **prometheus-nvidiasmi** - Needs GPU access
- **homeassistant** - Needs hardware device access

## Restart Policies

All services use `restart: unless-stopped`:
- Containers restart automatically after crashes
- Containers don't restart if manually stopped
- Containers restart after host reboot

## Update Strategy

**Watchtower** handles automatic updates:
- Checks for updates daily (86400 seconds)
- Automatically pulls new images and recreates containers
- Cleans up old images with `--cleanup` flag
- Configuration: `/opt/docker/watchtower/config.json`

To exclude a service from auto-updates:
```yaml
labels:
  - "com.centurylinklabs.watchtower.enable=false"
```

## Maintenance Commands

### Update all services manually
```bash
for dir in */; do
  cd "$dir"
  docker compose pull
  docker compose up -d
  cd ..
done
```

### Check service health
```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
```

### View logs
```bash
docker compose -f <service>/docker-compose.yml logs -f --tail=100
```

### Restart a service
```bash
cd <service>
docker compose restart <container-name>
```
