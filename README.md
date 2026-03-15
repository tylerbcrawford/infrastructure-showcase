# Self-Hosted Media & Infrastructure Stack

**51 services (46 Docker + 5 systemd) | 20TB Storage | Ubuntu Server**

## Why This Exists

I started self-hosting with Plex and a couple of *arr services. Then I needed subtitles, so I built [Subgeneratorr](https://github.com/tylerbcrawford/subgeneratorr). Then notifications needed rebranding, so I built [Boo Bot](https://github.com/tylerbcrawford/boo-bot). Then I needed backups, monitoring, book management for two users, a Discord-to-SMS bridge, and suddenly I was managing 51 services.

This repo documents the full stack — not as a tutorial, but as a reference for how all the pieces fit together. If you're building something similar, the architecture decisions and automation schedules might save you some time.

## Stack at a Glance

| Category | Services | Count |
|----------|----------|-------|
| **Core Media** | Plex, Sonarr, Radarr, Prowlarr, Bazarr, Lidarr, NZBGet | 7 |
| **Books** | 4× Readarr, 2× Calibre, 2× Calibre-Web, Audiobookshelf | 9 |
| **Downloads** | qBittorrent-VPN, qBittorrent-MAM, Tdarr, MeTube, Unpackerr | 5 |
| **AI** | Subgeneratorr (web + worker + Redis + CLI) | 4 |
| **Management** | Dashboard, Portainer, Tautulli, Glances, FileBrowser, WeTTY, Firecrawl UI, Watchtower | 8 |
| **Notifications** | Notifiarr, Trailarr, Pulsarr, Discord Bot, Tautulli-Digest, Webhook-Proxy, Twilio SMS, Landing Pages | 8 |
| **Infrastructure** | OAuth2 Proxy, Agent Dispatcher, Personal Finance | 3 |
| **Systemd** | Calibre auto-add (×2), Sonarr season limiter, ISO converter, Agent Dispatcher runner | 5 |

**Total: 51** (46 Docker + 5 systemd)

## Architecture Highlights

### Media Acquisition Pipeline

```
Prowlarr (indexers) → Sonarr/Radarr → NZBGet/qBittorrent → Unpackerr → Plex
                                                                 |
                                                  Bazarr (subtitles every 6h)
                                                                 |
                                                  Fallback → Subgeneratorr (AI)
```

### Book Pipeline

```
Readarr → NZBGet/qBittorrent-MAM → Unpackerr
                                        |
              calibre-autoadd-watcher (systemd) → Calibre auto-import → Calibre-Web
```

### Plex Watchlist Sync

```
Plex Watchlist → Pulsarr → Sonarr/Radarr
                                |
              sonarr-season-limiter (systemd) → limits new shows to Season 1
```

### Network & Auth

- Single Docker bridge network connecting all 46 containers
- nginx reverse proxy (systemd) with wildcard SSL via certbot
- Google OAuth2 Proxy protecting all web UIs — one login covers everything
- Exceptions: Kobo sync (token auth), Twilio webhook (HMAC-SHA1), Plex (own auth)

## Interesting Decisions

**Why 4 Readarr instances?** Two users, each needing separate ebook and audiobook libraries with independent quality profiles and root folders. Readarr doesn't support multi-user natively, so separate instances was the cleanest solution.

**Why a systemd season limiter?** Pulsarr auto-adds shows from Plex watchlists, but by default it monitors all seasons. For a new show, I only want Season 1 until I know I like it. A small systemd service watches Sonarr's API and flips new additions to Season 1 only.

**Why both Bazarr and Subgeneratorr?** Bazarr handles ~85% of subtitle needs from community sources. Subgeneratorr fills the gaps with AI transcription — obscure shows, older content, anything without community subs. They work in sequence, not in competition.

**Tiered startup** — services boot in 5 waves to prevent CPU spikes. Infrastructure first, then downloads, then media managers, then Plex, then notifications. Each wave waits for health checks before starting the next.

## Automation

Highlights from the cron schedule (see [docs/automation.md](docs/automation.md) for the full list):

| Schedule | Task |
|----------|------|
| Every 5 min | Dynamic DNS update |
| Every 6 hours | Health check (all services) |
| 3:00 AM daily | Incremental backup → Google Drive |
| 4:00 AM daily | Watchtower container updates |
| 9:00 AM daily | Plex stats digest to Discord |
| Sunday 2:00 AM | Forced full backup |
| Continuous | Calibre auto-add watchers, season limiter |

## Related Repos

| Repo | What it does |
|------|-------------|
| [restic-backup-system](https://github.com/tylerbcrawford/restic-backup-system) | Encrypted backups with Docker volume support and GDrive sync |
| [server-monitoring-suite](https://github.com/tylerbcrawford/server-monitoring-suite) | Health monitoring with Discord alerts and cooldown logic |
| [homelab-scripts](https://github.com/tylerbcrawford/homelab-scripts) | Cron scripts for metadata enrichment, ebook management, and more |
| [subgeneratorr](https://github.com/tylerbcrawford/subgeneratorr) | AI subtitle generation (Deepgram Nova-3) |
| [boo-bot](https://github.com/tylerbcrawford/boo-bot) | Discord bot for media server management |

## Documentation

Detailed reference docs live in `docs/`:
- **[Architecture](docs/architecture.md)** — full service topology and data flows
- **[Security](docs/security.md)** — hardening details, OAuth2 flow, credential management
- **[Automation](docs/automation.md)** — complete cron schedule and monitoring setup

## License

[MIT](LICENSE) — Copyright 2026 Tyler Crawford
