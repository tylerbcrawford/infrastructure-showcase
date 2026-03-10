# Architecture Overview

## Service Topology

This infrastructure runs **51 services** organized into functional categories:

| Category | Docker | Systemd | Total |
|----------|--------|---------|-------|
| Core Media (Plex, *arr suite) | 7 | 0 | 7 |
| Book Management | 9 | 2 | 11 |
| Download Clients | 4 | 0 | 4 |
| AI Services | 4 | 0 | 4 |
| Management & Monitoring | 10 | 1 | 11 |
| Notifications & Integration | 7 | 1 | 8 |
| Landing Pages | 2 | 0 | 2 |
| Authentication | 1 | 0 | 1 |
| Other (Tdarr, Watchtower, IRC) | 3 | 1 | 4 |
| **Total** | **46** | **5** | **51** |

## Network Design

### Single Bridge Network

All 46 Docker containers share a single bridge network (`plex_network`) with subnet `172.20.0.0/16`. This keeps inter-service communication simple -- any container can reach any other by hostname.

```
Internet
    |
    v
[nginx reverse proxy] (systemd, ports 80/443)
    |
    +---> [OAuth2 Proxy :4180] ---> Google OAuth2
    |
    +---> [plex :32400]
    +---> [sonarr :8989]
    +---> [radarr :7878]
    +---> ... (all services on plex_network)
```

### Why a single network?

- **Simplicity over isolation**: With 46 containers needing to talk to each other (Sonarr to Prowlarr, Unpackerr to all *arr services, Notifiarr to everything), multi-network setups create more complexity than security benefit.
- **OAuth2 provides the security boundary**: All external access goes through nginx + OAuth2 Proxy. The Docker network is internal-only.
- **Static gateway IP** (`172.20.0.1`): Used by WeTTY to SSH back to the host via `host.docker.internal`.

### Reverse Proxy Architecture

nginx runs as a **systemd service** (not Docker) for stability -- it's the entry point for all external traffic:

1. **Wildcard DNS**: `*.yourdomain.com` points to the server
2. **Wildcard SSL**: Single certbot certificate covers all subdomains
3. **Per-service vhosts**: Each service gets its own nginx config in `/etc/nginx/sites-available/`
4. **OAuth2 subrequest**: Each vhost includes an `auth_request` block that validates the session with OAuth2 Proxy before proxying to the upstream service

### VPN Isolation

The `qbittorrent-vpn` container uses `binhex/arch-qbittorrentvpn` which:

- Runs OpenVPN inside the container
- Has a kill switch (`STRICT_PORT_FORWARD=yes`) -- if VPN drops, all traffic stops
- Exposes Privoxy (port 8118) for other containers to route through the VPN
- Also exposes the Mousehole VPN leak detection port (5010)

The second qBittorrent instance (`qbittorrent-mam`) runs without VPN for private tracker use where VPN is not required.

## Storage Architecture

### Two-Drive Layout

| Drive | Mount | Purpose |
|-------|-------|---------|
| System SSD | `/` | OS, Docker images, named volumes (service configs) |
| 20TB HDD | `/path/to/media/` | All media files, downloads, books |

### Named Volumes vs Bind Mounts

- **Named volumes** (`sonarr_config:`, `plex_config:`, etc.): Used for service configuration data. Managed by Docker, backed up separately.
- **Bind mounts** (`${MEDIA_PATH}/tv:/tv`): Used for media files. Shared across multiple containers that need access to the same data.

### Media Directory Structure

```
media/
├── tv/              # Sonarr manages, Plex reads
├── movies/          # Radarr manages, Plex reads
├── movies-4k/       # Radarr manages, Tdarr transcodes
├── music/           # Lidarr manages
├── downloads/       # Staging area (NZBGet + qBittorrent write, *arr services import from)
│   ├── completed/
│   ├── movies/
│   ├── tv/
│   ├── tv-sonarr/
│   ├── torrents/
│   └── music/
├── books/
│   ├── user1/
│   │   ├── ebooks/      # Readarr output
│   │   ├── audiobooks/  # Readarr-Audio output
│   │   └── calibre/     # Calibre library + autoadd/
│   └── user2/           # Same structure
├── audiobooks/          # Audiobookshelf serves
└── YouTube/_INBOX/      # MeTube downloads
```

## Service Dependencies

The `docker-compose.yml` uses health checks and `depends_on` with `condition: service_healthy` to enforce startup order:

```
prowlarr (must be healthy)
    |
    +--> sonarr
    +--> radarr
    +--> readarr (x4)

nzbget (must be healthy)
    |
    +--> sonarr
    +--> radarr

qbittorrent-vpn (must be healthy)
    |
    +--> sonarr
    +--> radarr

plex (must be healthy)
    |
    +--> notifiarr

subgeneratorr-redis (must be healthy)
    |
    +--> subgeneratorr-web
    +--> subgeneratorr-worker

readarr + sonarr + radarr (must be healthy)
    |
    +--> discord-bot
```

## Resource Allocation

Services are constrained with CPU and memory limits via `deploy.resources.limits`:

| Tier | Memory | CPU | Services |
|------|--------|-----|----------|
| Heavy | 4-8G | 1.5-3.0 | Plex, Tdarr |
| Medium | 1-2G | 1.0-1.5 | Sonarr, Radarr, NZBGet, qBittorrent-VPN, Calibre x2, Trailarr, Audiobookshelf, Subgeneratorr Worker |
| Standard | 256-512M | 0.5 | Prowlarr, Bazarr, Lidarr, Readarr x4, Calibre-Web x2, Pulsarr, qBit-MAM, All remaining |
| Light | 64-128M | 0.1-0.25 | Landing pages, WeTTY, Webhook-proxy, Twilio-SMS, MAM-IRC, Watchtower, Glances, Firecrawl-UI |

Total memory budget: ~30-35GB across all containers (designed for a 64GB RAM server).
