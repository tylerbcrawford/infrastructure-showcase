# My Homelab: 57 Services, One Tailnet

**57 services (51 Docker + 6 systemd) | 20TB Storage | Ubuntu Server | 6-node Tailscale fleet**

## Why This Exists

I started self-hosting with Plex and a couple of *arr services. Then I needed subtitles, so I built [Subgeneratorr](https://github.com/tylerbcrawford/subgeneratorr). Then notifications needed rebranding, so I built [Boo Bot](https://github.com/tylerbcrawford/boo-bot). Then I needed backups, monitoring, book management for two users, a Discord-to-SMS bridge, and suddenly I was managing 57 services across a home server, a VPS, and a tailnet that ties them together.

This is my homelab: a home server, a VPS, and a tailnet that ties them together, built one itch at a time. This repo documents the full stack as a reference for how all the pieces fit together. If you're building something similar, the architecture decisions and automation schedules might save you some time.

## Stack at a Glance

| Category | Services | Count |
|----------|----------|-------|
| **Core Media** | Plex, Sonarr, Radarr, Prowlarr, Bazarr, Lidarr, NZBGet | 7 |
| **Books** | 4× Readarr, 2× Calibre, 2× Calibre-Web, Audiobookshelf, Abook (EPUB→audiobook), rreading-glasses + its Postgres | 12 |
| **Downloads** | qBittorrent-VPN, qBittorrent-MAM, MAM-IRC, Tdarr, MeTube, Unpackerr | 6 |
| **AI & Search** | Subgeneratorr (web + worker + Redis), Crawl4AI, SearXNG + Valkey | 6 |
| **Management** | Dashboard, Portainer, Tautulli, Glances, FileBrowser, WeTTY, WUD, PR Watcher | 8 |
| **Notifications & Integration** | Notifiarr, Trailarr, Tunarr, Pulsarr, Discord Bot, Tautulli-Digest, Webhook-Proxy, Twilio SMS, Landing Pages (×2) | 10 |
| **Infrastructure** | OAuth2 Proxy, SOCKS5 egress (tailnet-only) | 2 |
| **Systemd** | Calibre auto-add (×2), Sonarr season limiter, PDF/EPUB converter (×2), Watchlistarr health check | 6 |

**Total: 57** (51 Docker + 6 systemd)

## Security Posture

Every web UI sits behind a single Google OAuth2 gateway, so one login covers the whole stack and there are no per-service passwords to hand out.

```
User --> nginx --> auth_request to OAuth2 Proxy
                       |
                       +--> Not authenticated? Redirect to Google login
                       |
                       +--> Authenticated? Proxy to upstream service
```

Hardening layered underneath:

- **TLS everywhere** — one wildcard Let's Encrypt cert, HTTP redirected to HTTPS, HSTS with preload.
- **UFW default-deny inbound**; only 80, 443, and Plex's direct port are open to the internet. SSH is not: port 22 is allowed only from the tailnet and one peer server, so there is no public SSH surface to brute-force.
- **fail2ban** jails on SSH and nginx auth failures (3–5 retries, 1-hour bans).
- **SSH is key-only**, root login disabled, `MaxAuthTries 3`, and reachable only over Tailscale.
- **Per-container resource limits** so no single container can starve the host, with read-only media mounts and a locked-down Docker socket.
- **VPN kill switch** on the primary torrent client: if the tunnel drops, `STRICT_PORT_FORWARD` halts all traffic.

Full details, including the email allowlist, sysctl tuning, and unattended security updates, are in [docs/security.md](docs/security.md).

## Architecture Highlights

The full service graph, ports, and data flows are drawn out in the [service topology diagram](diagrams/service-topology.md).

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

- Single Docker bridge network connecting all 51 containers
- nginx reverse proxy (systemd) with wildcard SSL via certbot's DNS challenge
- Google OAuth2 Proxy protecting all web UIs — one login covers everything
- Exceptions: Kobo sync (token auth), Twilio webhook (HMAC-SHA1), Plex (own auth)
- Anything not meant for the internet is bound to the machine's Tailscale address instead of `0.0.0.0` — see below

### Remote Access & Tailnet

The home server is one node of a six-node [Tailscale](https://tailscale.com) tailnet: two Linux servers (this one and a public VPS), two Macs, a Linux laptop, and an iPhone. WireGuard tunnels between them, MagicDNS for names, no port-forwarding for anything that isn't a public web UI.

| Concern | How it's handled |
|---|---|
| **SSH** | OpenSSH listens, but UFW only accepts port 22 from the tailnet. `ssh media` from any fleet machine resolves through MagicDNS and rides the WireGuard tunnel; nothing is exposed on the WAN. |
| **Private services** | Crawl4AI (`:11235`), the SOCKS5 egress (`:1080`), and RDP (`:3389`) are bound to the `100.x` Tailscale address or allowed only `on tailscale0`. The VPS reaches them by tailnet IP; the internet can't see them. |
| **Cross-machine sharing** | The VPS runs no scraper of its own: its agents call this server's Crawl4AI over the tailnet. The grocery pipeline routes through this server's residential IP via the tailnet SOCKS proxy. |
| **Split access model** | Public: `https://service.domain` → nginx → OAuth2 → upstream. Private: `http://100.x.y.z:port` over Tailscale, no OAuth needed because the tunnel *is* the auth. |
| **Key expiry** | Always-on nodes have key expiry disabled; laptops and the phone keep the 90-day default. A [fleet watchdog](https://github.com/tylerbcrawford/tailscale-fleet-watchdog) alerts on daemon logout, imminent expiry, expiry-config drift, and offline always-on nodes — built after a silent key expiry took the server off the tailnet. |

Full DNS, addressing, firewall and routing detail: [docs/networking.md](docs/networking.md).

## Interesting Decisions

**Why 4 Readarr instances?** Two users, each needing separate ebook and audiobook libraries with independent quality profiles and root folders. Readarr doesn't support multi-user natively, so separate instances was the cleanest solution.

**Why a systemd season limiter?** Pulsarr auto-adds shows from Plex watchlists, but by default it monitors all seasons. For a new show, I only want Season 1 until I know I like it. A small systemd service watches Sonarr's API and flips new additions to Season 1 only.

**Why both Bazarr and Subgeneratorr?** Bazarr handles ~85% of subtitle needs from community sources. Subgeneratorr fills the gaps with AI transcription — obscure shows, older content, anything without community subs. They run in sequence: Bazarr first, then Subgeneratorr on what's left.

**Tiered startup** — services boot in 5 waves to prevent CPU spikes. Infrastructure first, then downloads, then media managers, then Plex, then notifications. Each wave waits for health checks before starting the next.

## Automation

Highlights from the cron schedule (see [docs/automation.md](docs/automation.md) for the full list):

| Schedule | Task |
|----------|------|
| Every 5 min | Dynamic DNS update |
| Every 6 hours | Health check (all services) |
| 3:00 AM daily | Incremental backup → Google Drive |
| 9:00 AM daily | Plex stats digest to Discord |
| Daily 8:00 AM | Tailscale self-check (daemon state, key expiry) |
| Thursday 8:00 AM | WUD update scan → tiered auto-updater |
| Sunday 8:30 AM | Tailscale fleet audit via REST API |
| Sunday 2:00 AM | Forced full backup |
| Continuous | Calibre auto-add watchers, season limiter |

## Related Repos

| Repo | What it does |
|------|-------------|
| [tailscale-fleet-watchdog](https://github.com/tylerbcrawford/tailscale-fleet-watchdog) | Node-key expiry and fleet-health watchdog for the tailnet |
| [restic-backup-system](https://github.com/tylerbcrawford/restic-backup-system) | Encrypted backups with Docker volume support and offsite sync (Google Drive / Backblaze B2) |
| [server-monitoring-suite](https://github.com/tylerbcrawford/server-monitoring-suite) | Health monitoring with Discord alerts and cooldown logic |
| [homelab-scripts](https://github.com/tylerbcrawford/homelab-scripts) | Cron scripts for metadata enrichment, ebook management, and more |
| [subgeneratorr](https://github.com/tylerbcrawford/subgeneratorr) | AI subtitle generation (Deepgram Nova-3) |
| [boo-bot](https://github.com/tylerbcrawford/boo-bot) | Discord bot for media server management |

## Documentation

Detailed reference docs live in `docs/`:
- **[Architecture](docs/architecture.md)** — full service topology and data flows
- **[Service Topology Diagram](diagrams/service-topology.md)** — Mermaid graph of every service, port, and data flow
- **[Networking](docs/networking.md)** — DNS layers, address spaces, firewall rules, traffic paths, tailnet
- **[Security](docs/security.md)** — hardening details, OAuth2 flow, credential management
- **[Automation](docs/automation.md)** — complete cron schedule and monitoring setup

## License

[MIT](LICENSE) — Copyright 2026 Tyler Crawford
