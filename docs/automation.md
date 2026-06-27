# Automation & Monitoring

## Overview

The infrastructure uses a layered automation approach:

1. **Docker-level**: Watchtower for container updates, health checks for auto-restart
2. **Cron-level**: 15+ scheduled scripts for maintenance, cleanup, and monitoring
3. **Systemd-level**: 6 persistent services for real-time file watchers and scheduled converters
4. **External tools**: Dedicated repos for backup, monitoring, and scripting

## Companion Repositories

| Repository | Purpose | Link |
|------------|---------|------|
| **restic-backup-system** | Encrypted, versioned backups with Discord notifications | [GitHub](https://github.com/tylerbcrawford/restic-backup-system) |
| **server-monitoring-suite** | Health monitoring, service checks, alerting | [GitHub](https://github.com/tylerbcrawford/server-monitoring-suite) |
| **homelab-scripts** | Automation scripts for media server management | [GitHub](https://github.com/tylerbcrawford/homelab-scripts) |

## Container Auto-Updates (Watchtower)

Watchtower runs daily at **4:00 AM** and updates containers that opt in via labels:

```yaml
watchtower:
  environment:
    - WATCHTOWER_SCHEDULE=0 0 4 * * *
    - WATCHTOWER_CLEANUP=false
    - WATCHTOWER_ROLLING_RESTART=true
```

- **Rolling restart**: Updates containers one at a time to minimize downtime
- **Selective updates**: Custom-built containers (webhook-proxy, twilio-sms, subgeneratorr, etc.) are excluded via `com.centurylinklabs.watchtower.enable=false`
- **Discord notifications**: Watchtower posts update summaries to a Discord webhook

## Cron Job Portfolio

### Every 5 Minutes

| Job | Purpose |
|-----|---------|
| Dynamic DNS update | Keep domain pointing to current IP (Dynu provider) |

### Every 30 Minutes

| Job | Purpose |
|-----|---------|
| Nested archive extraction | Extract ZIP/RAR files downloaded by Readarr |
| EPUB sync from GDrive | Pull converted EPUBs from Google Drive |

### Daily (Overnight)

| Time | Job | Purpose |
|------|-----|---------|
| 2:00 AM | Subtitle sync | Synchronize AI-generated subtitles |
| 2:00 AM | Missing books list update | Track unavailable books across Readarr instances |
| 3:00 AM | Incremental backup | Backup configs if changes detected |
| 3:00 AM | SSL renewal check | Certbot auto-renewal |
| 3:05-3:20 AM | Readarr cleanup (x4) | Staggered cleanup across 4 Readarr instances |
| 4:00 AM | Container updates | Watchtower checks for new images |
| 9:00 AM | Plex stats digest | Daily viewing stats posted to Discord |

### Weekly

| Day/Time | Job | Purpose |
|----------|-----|---------|
| Sunday 00:00 | Log compression | gzip logs >5MB, delete .gz >30 days |
| Sunday 00:30 | Readarr log cleanup | Delete logs >14 days |
| Sunday 2:00 AM | Forced backup | Full backup regardless of changes |
| Sunday 5:00 AM | Intermediate folder cleanup | Remove empty staging directories |
| Sunday 5:00 AM | Audiobookshelf author match | Update author metadata |
| Monday 6:00 AM | Log health check | Audit all log directories for anomalies |

### Monthly

| Day/Time | Job | Purpose |
|----------|-----|---------|
| 1st, 9:00 AM | Indexer analytics | Download statistics by indexer |

## Systemd Services

### calibre-autoadd-watcher (x2 instances)

Watches Readarr's ebook output directories and copies new files to Calibre's auto-add folder for automatic library ingestion.

```
Readarr downloads ebook --> /books/user1/ebooks/
                                    |
            calibre-autoadd-watcher detects new file
                                    |
                  Copies to /books/user1/calibre/autoadd/
                                    |
                      Calibre auto-imports to library
                                    |
                        Calibre-Web serves to reader
```

### sonarr-season-limiter

Monitors Pulsarr-added TV shows and limits them to Season 1 only. Prevents watchlist additions from downloading entire series backlogs.

## Health Check Architecture

### Docker Health Checks

18 containers have built-in health checks:

- **HTTP probes**: `curl -fsS http://localhost:PORT/endpoint` for web services
- **Process checks**: `pgrep -f process_name` for background daemons
- **Custom checks**: Redis uses `redis-cli ping | grep PONG`

Health check results enable:

1. `depends_on: condition: service_healthy` for startup ordering
2. Automatic container restart via `restart: on-failure`
3. Status visibility in `docker compose ps`

### Log Health Check Script

A weekly script audits all log directories:

- Docker container JSON logs (size and count)
- Journald usage (against 500MB limit)
- Custom script log directories (warns if >30 files)
- *arr application internal logs
- Stray log files in unexpected locations

## Backup System

The backup system is maintained in a separate repository: [restic-backup-system](https://github.com/tylerbcrawford/restic-backup-system).

### What Gets Backed Up

- Docker Compose configuration
- Environment variables (`.env`)
- Service configuration files
- Automation scripts
- nginx configurations

### Schedule

| Frequency | Time | Type |
|-----------|------|------|
| Daily | 3:00 AM | Incremental (if changes) |
| Weekly | Sunday 2:00 AM | Forced full |

### Features

- **Encrypted**: All backups encrypted at rest
- **Versioned**: Restic deduplication and snapshots
- **Discord notifications**: Success/failure alerts
- **Offsite**: Backed up to remote storage

## Log Rotation

### Logrotate

```
# /etc/logrotate.d/mediaserver-custom
weekly, rotate 4, compress, copytruncate
```

The `su user user` directive is required because parent directories have group-write permissions. Without it, logrotate silently skips all files.

### Docker Daemon

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

### Journald

```ini
# /etc/systemd/journald.conf
SystemMaxUse=500M
```
