# Self-Hosted Media & Infrastructure Stack

**51 services (46 Docker + 5 systemd) | 20TB Storage | Ubuntu Server**

A production self-hosted media server infrastructure running 51 services across Docker Compose and systemd. Features automated media acquisition, book management, AI-powered subtitle generation, comprehensive monitoring, and encrypted backups.

## Quick Stats

| Metric | Value |
|--------|-------|
| Total Services | 51 (46 Docker + 5 systemd) |
| Storage | 20TB (single drive) |
| Reverse Proxy | nginx with wildcard SSL |
| Authentication | Google OAuth2 (all web UIs) |
| Monitoring | Glances + Tautulli + custom health checks |
| Backups | Restic (encrypted, versioned, offsite) |
| OS | Ubuntu Server |

---

## Service Overview

### Core Media Services

| Service | Port | Role |
|---------|------|------|
| Plex | 32400 | Media streaming server |
| Sonarr | 8989 | TV show management & automation |
| Radarr | 7878 | Movie management & automation |
| Prowlarr | 9696 | Indexer management (feeds Sonarr/Radarr/Readarr) |
| Bazarr | 6767 | Subtitle management (automated downloads every 6h) |
| Lidarr | 8686 | Music management & automation |
| NZBGet | 6789 | Usenet download client |

### Book Management (4 Readarr + 2 Calibre + 2 Calibre-Web + 1 Audiobookshelf)

| Service | Port | Role |
|---------|------|------|
| Readarr | 8787 | User 1 ebook acquisition (Bookshelf fork) |
| Readarr-Audio | 8788 | User 1 audiobook acquisition |
| Readarr2 | 8789 | User 2 ebook acquisition |
| Readarr-Audio2 | 8790 | User 2 audiobook acquisition |
| Calibre | 8083/8084 | User 1 library management (web + VNC) |
| Calibre2 | 8085/8086 | User 2 library management (web + VNC) |
| Calibre-Web | 8087 | User 1 web reader (+ Kobo sync) |
| Calibre-Web2 | 8088 | User 2 web reader |
| Audiobookshelf | 13378 | Audiobook/podcast streaming server |

### Download Services

| Service | Port | Role |
|---------|------|------|
| qBittorrent-VPN | 8090 | Torrent client (VPN-enforced, kill switch) |
| qBittorrent-MAM | 8092 | Dedicated private tracker client |
| MAM-IRC | - | IRC idler for private tracker bonus points |
| Tdarr | 8265 | Video transcoding automation |
| MeTube | 8401 | YouTube/web video downloader |
| Unpackerr | - | Automatic archive extraction for *arr services |

### AI Services

| Service | Port | Role |
|---------|------|------|
| Subgeneratorr Web | 5000 | AI subtitle generation UI (Deepgram Nova-3) |
| Subgeneratorr Worker | - | Celery async transcription worker |
| Subgeneratorr Redis | 6379 | Job queue backend |
| Subgeneratorr CLI | - | Batch CLI subtitle generation |

### Management & Monitoring

| Service | Port | Role |
|---------|------|------|
| Custom Dashboard | 3400 | Centralized service dashboard |
| Portainer | 9444 | Docker container management |
| Tautulli | 8182 | Plex analytics & statistics |
| Glances | 8383 | Real-time system monitoring |
| FileBrowser | 8089 | Web-based file management |
| WeTTY | 3002 | Browser-based SSH terminal |
| Firecrawl UI | 8093 | Web scraping interface |
| Watchtower | - | Automatic container updates (daily 4 AM) |
| Agent Dispatcher | 3100 | AI agent orchestration dashboard |
| Personal Finance | 3200 | Self-hosted finance dashboard |

### Notifications & Integration

| Service | Port | Role |
|---------|------|------|
| Notifiarr | 5454 | Discord notification relay for all *arr services |
| Trailarr | 7889 | Automatic trailer downloads for movies/TV |
| Pulsarr | 3003 | Plex watchlist sync to Sonarr/Radarr (real-time) |
| Discord Bot | - | Custom bot (6 cogs: rebranding, suppression, management) |
| Tautulli-Digest | 8078 | Daily Plex stats digest to Discord (9 AM) |
| Webhook-Proxy | 8079 | Custom webhook relay |
| Twilio SMS | 3300 | Inbound SMS/MMS to Discord relay (HMAC-SHA1 auth) |
| Landing Pages | 8080/8081 | Static nginx landing pages |

### Systemd Services (5)

| Service | Role |
|---------|------|
| calibre-autoadd-watcher | Monitors Readarr output, auto-imports to Calibre (User 1) |
| calibre-autoadd-watcher-user2 | Same for User 2 |
| sonarr-season-limiter | Limits Pulsarr-added shows to Season 1 only |
| iso-converter-watcher | ISO to MKV conversion (disabled) |
| agent-dispatcher | AI agent runner (node-pty, Unix socket to web container) |

---

## Architecture

### Data Flow: Media Acquisition Pipeline

```
Prowlarr (indexers) --> Sonarr/Radarr --> NZBGet/qBittorrent --> Unpackerr --> Plex
                                                                    |
                                                    Bazarr (subtitles every 6h)
                                                                    |
                                                    If not found --> Subgeneratorr (AI)
```

### Data Flow: Book Management Pipeline

```
Readarr --> NZBGet/qBittorrent-MAM --> Unpackerr extracts
                                            |
              calibre-autoadd-watcher (systemd) monitors /ebooks folder
                                            |
                        Copies to /calibre/autoadd folder
                                            |
                  Calibre auto-imports --> Calibre-Web serves
```

### Data Flow: Plex Watchlist Sync

```
Plex Watchlist --> Pulsarr --> Sonarr/Radarr (adds shows/movies)
                                            |
              sonarr-season-limiter (systemd) limits new shows to Season 1
```

### Network Architecture

- **Single Docker bridge network** (`plex_network`) connecting all 46 containers
- **nginx reverse proxy** (systemd, not Docker) handles all external HTTPS traffic
- **Wildcard SSL** certificate via certbot for `*.yourdomain.com`
- **OAuth2 Proxy** (Google provider) protects all web UIs with a single sign-on flow
- Services communicate internally via Docker DNS hostnames (e.g., `sonarr:8989`, `radarr:7878`)

### Storage Layout

```
/path/to/media/                          # 20TB drive
├── tv/                                  # Sonarr --> Plex
├── movies/                              # Radarr --> Plex
├── downloads/                           # Staging area
│   ├── movies/tv/tv-sonarr/torrents/
│   └── podcasts/
├── books/
│   ├── user1/
│   │   ├── ebooks/                      # Readarr output
│   │   ├── audiobooks/                  # Readarr-Audio output
│   │   └── calibre/                     # Calibre library + /autoadd
│   └── user2/
│       ├── ebooks/calibre/audiobooks/   # Same structure
├── audiobooks/                          # Audiobookshelf
└── YouTube/_INBOX/                      # MeTube downloads
```

### Docker Internal Hostnames

Services communicate over the Docker bridge network using these hostnames:

```bash
# Core Media
plex:32400  sonarr:8989  radarr:7878  prowlarr:9696  nzbget:6789  bazarr:6767  lidarr:8686

# Books (all Readarr instances use internal port 8787)
readarr:8787  readarr-audio:8787  readarr2:8787  readarr-audio2:8787
calibre:8080  calibre2:8080  calibre_web:8083  calibre_web2:8083  audiobookshelf:80

# Download & Processing
qbittorrent-vpn:8080  qbittorrent-mam:8080  tdarr:8265  metube:8081  unpackerr:N/A

# AI
subgeneratorr-web:5000  subgeneratorr-worker:N/A  subgeneratorr-redis:6379

# Management
dashboard:3400  portainer:9443  glances:61208  tautulli:8181
filebrowser:80  firecrawl-ui:8080  wetty:3000  agent-dispatcher-web:3100  personal-finance:3200

# Background
oauth2-proxy:4180  pulsarr:3003  notifiarr:5454  trailarr:7889
watchtower:N/A  webhook-proxy:N/A  twilio-sms:3300  discord-bot:N/A  landing-page:80
```

---

## Authentication & Security

All web-facing services are protected by a centralized **OAuth2 Proxy** (Google provider):

1. User visits `https://service.yourdomain.com`
2. nginx sends auth subrequest to OAuth2 Proxy
3. If unauthenticated, redirected to `https://auth.yourdomain.com` (Google login)
4. After authentication, cookie set on `*.yourdomain.com` (7-day expiry, 24h refresh)
5. Subsequent requests pass through — user identity forwarded to upstream services

**Exceptions** (public access, no OAuth2):
- Kobo e-reader sync endpoint (token-based authentication in URL)
- Twilio SMS webhook (HMAC-SHA1 signature verification)
- Plex (has its own authentication)

See [docs/security.md](docs/security.md) for full security hardening details.

---

## Automation Schedule

| Time | Task | Method |
|------|------|--------|
| Every 5 min | DNS IP update | Cron (Dynu) |
| Every 30 min | Extract nested archives | Cron script |
| Every 30 min | Sync converted EPUBs from GDrive | Cron script |
| Every 6 hours | Media server health check | Cron script |
| 2:00 AM | Weekly forced backup (Sundays) | Cron script |
| 2:00 AM | Subtitle sync | Cron script |
| 3:00 AM | Daily backup (if changes) | Cron script |
| 3:00 AM | SSL renewal check | Certbot cron |
| 3:05-3:20 AM | Readarr cleanup (4 instances, staggered) | Cron scripts |
| 4:00 AM | Container updates | Watchtower (Docker) |
| 5:00 AM | Intermediate folder cleanup (Sundays) | Cron script |
| 5:00 AM | Audiobookshelf author metadata | Cron script |
| 9:00 AM | Plex stats digest to Discord | tautulli-digest (Docker) |
| Weekly (Sun) | Log cleanup & rotation | Cron + logrotate |
| Weekly (Mon 6 AM) | Log health check | Cron script |
| Monthly (1st, 9 AM) | Indexer download analytics | Cron script |
| Continuous | Calibre auto-add (2 instances) | Systemd watchers |
| Continuous | Sonarr season limiter | Systemd service |

See [docs/automation.md](docs/automation.md) for details on monitoring and backup systems.

---

## Credential Management

All credentials are managed via a `.env` file in the project root. The `docker-compose.yml` references secrets exclusively through `${VARIABLE}` syntax. See [`.env.example`](.env.example) for the complete list of required environment variables.

**Categories of secrets managed:**
- OAuth2 provider credentials (Client ID, Client Secret, Cookie Secret)
- Service API keys (*arr services, Plex, Tautulli, Notifiarr, etc.)
- Download client credentials (NZBGet, Usenet provider)
- External API keys (Deepgram, Anthropic, OpenAI, TMDB, Firecrawl)
- Discord bot token and webhook URLs
- Twilio credentials (Account SID, Auth Token)
- VPN credentials

---

## nginx Configuration

### Template: OAuth2-Protected Service

```nginx
server {
    listen 80;
    server_name SERVICENAME.yourdomain.com;
    if ($scheme = http) { return 301 https://$host$request_uri; }
    listen 443 ssl http2;
    include /etc/nginx/snippets/ssl-yourdomain.com.conf;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    location = /oauth2/auth {
        proxy_pass http://127.0.0.1:4180/oauth2/auth;
        proxy_pass_request_body off;
        proxy_set_header Content-Length "";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Uri $request_uri;
        proxy_set_header X-Auth-Request-Redirect $request_uri;
        proxy_set_header Authorization "";
    }

    location @oauth2_signin {
        return 302 https://auth.yourdomain.com/oauth2/start?rd=$scheme://$host$request_uri;
    }

    location / {
        auth_request /oauth2/auth;
        error_page 401 = @oauth2_signin;
        error_page 403 = @oauth2_signin;

        auth_request_set $auth_email $upstream_http_x_auth_request_email;
        auth_request_set $auth_user $upstream_http_x_auth_request_user;
        auth_request_set $auth_jwt $upstream_http_authorization;

        proxy_set_header X-Auth-Request-Email $auth_email;
        proxy_set_header X-Forwarded-User $auth_user;
        proxy_set_header Authorization $auth_jwt;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_pass http://127.0.0.1:PORT;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        client_max_body_size 50m;
    }
}

map $http_upgrade $connection_upgrade {
    default upgrade;
    "" close;
}
```

### Template: Public Access (No OAuth2)

```nginx
server {
    listen 80;
    server_name SERVICENAME.yourdomain.com;
    if ($scheme = http) { return 301 https://$host$request_uri; }
    listen 443 ssl http2;
    include /etc/nginx/snippets/ssl-yourdomain.com.conf;
    add_header X-Robots-Tag "noindex, nofollow" always;

    location / {
        proxy_pass http://127.0.0.1:PORT;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## Log Management

### Log Directories

| Directory | Purpose | Rotation |
|-----------|---------|----------|
| `~/.local/share/mediaserver-logs/` | Active cron output (health checks, cleanup, extraction) | logrotate weekly + cron gzip/delete |
| `project-root/logs/` | Script startup logs | logrotate weekly |
| `project-root/missing-readarr/logs/` | Daily sync logs (dated files) | logrotate weekly + cron delete >14d |
| Docker container logs | JSON logs in `/var/lib/docker/containers/` | daemon.json: 10MB x 3 files per container |
| Journald | systemd service logs | 500MB max (`SystemMaxUse=500M`) |

### Docker Log Limits

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

### Logrotate Config

```
/path/to/media-server/logs/*.log
/path/to/media-server/missing-readarr/logs/*.log
~/.local/share/intermediate-cleanup/logs/*.log
~/.local/share/mediaserver-logs/*.log {
    su user user
    weekly
    rotate 4
    compress
    missingok
    notifempty
    copytruncate
}
```

---

## Cookbook: Adding a New Service

### Step 1: Add to docker-compose.yml

```yaml
new-service:
  image: vendor/service:latest
  container_name: new-service
  environment:
    - PUID=${PUID:-1000}
    - PGID=${PGID:-1000}
    - TZ=${TZ:-America/Toronto}
  volumes:
    - new_service_config:/config
  ports:
    - "NEW_PORT:INTERNAL_PORT"
  networks:
    - plex_network
  restart: unless-stopped
```

### Step 2: Create nginx config

```bash
sudo nano /etc/nginx/sites-available/newservice-subdomain
# Use the OAuth2-protected template above, replacing SERVICENAME and PORT
```

### Step 3: Enable and test

```bash
sudo ln -sf /etc/nginx/sites-available/newservice-subdomain /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
curl -I https://newservice.yourdomain.com  # Should redirect to OAuth2
```

### Step 4: Start service

```bash
cd /path/to/media-server
docker compose up -d new-service
```

---

## Troubleshooting

### Container Won't Start

```bash
docker compose logs <service-name>
docker compose restart <service-name>
docker compose up -d --force-recreate <service-name>
```

### VPN Down

```bash
# Check VPN IP (should NOT show your local IP)
docker exec qbittorrent-vpn curl -s checkip.amazonaws.com

# Restart VPN container
docker compose restart qbittorrent-vpn

# Check VPN logs
docker logs qbittorrent-vpn | grep -i "initialization sequence completed"
```

### OAuth2 Issues

```bash
docker compose logs oauth2-proxy
sudo nginx -t && sudo systemctl reload nginx

# Test OAuth2 redirect (should show HTTP 302)
curl -I https://sonarr.yourdomain.com
```

### Permission Issues

```bash
sudo chown -R 1000:1000 /path/to/media/
```

---

## Related Repositories

| Repository | Description |
|------------|-------------|
| [restic-backup-system](https://github.com/tylerbcrawford/restic-backup-system) | Encrypted, versioned backup system with Discord notifications |
| [server-monitoring-suite](https://github.com/tylerbcrawford/server-monitoring-suite) | Health monitoring, service checks, and alerting |
| [homelab-scripts](https://github.com/tylerbcrawford/homelab-scripts) | Automation scripts for media server management |

---

## Getting Started

1. **Clone this repo** and copy `.env.example` to `.env`
2. **Fill in credentials** in `.env` (OAuth2, API keys, passwords)
3. **Set up storage** mount points matching the volume paths in `docker-compose.yml`
4. **Configure nginx** using the templates above for each service subdomain
5. **Obtain SSL certificates** via certbot (`certbot certonly --dns-plugin ...` for wildcard)
6. **Start services** with `docker compose up -d`
7. **Set up systemd services** for calibre-autoadd-watcher, sonarr-season-limiter, agent-dispatcher

See [docs/architecture.md](docs/architecture.md) for the full service topology and [docs/security.md](docs/security.md) for hardening steps.

---

## License

[MIT](LICENSE) - Copyright 2026 Tyler Crawford
