# Networking

How traffic gets in, how machines find each other, and what is deliberately unreachable. Two servers are involved: the **home server** (residential connection, dynamic IP, behind a consumer router) and a **VPS** (static public IP). A Tailscale tailnet joins them to two Macs, a Linux laptop, and a phone.

## 1. DNS — three layers

| Layer | Names | Resolver | Notes |
|---|---|---|---|
| **Public** | `*.homedomain` → home server, `*.vpsdomain` → VPS | Registrar / DNS host | One wildcard `A` record per machine. The home record is kept current by a **dynamic-DNS updater every 5 minutes** because the residential IP changes. |
| **Tailnet (MagicDNS)** | `media-server`, `vps`, `mac-mini`, … `.tailnet.ts.net` | Tailscale's per-node DNS | Every node resolves every other by short name, on every network, including cellular. This is what `ssh media` actually uses. |
| **Container** | `sonarr`, `prowlarr`, `oauth2-proxy`, … | Docker's embedded DNS (`127.0.0.11`) | Containers on the shared bridge network resolve each other by service name. nginx on the host reaches them by published port instead. |

**Certificates** ride on the public layer: one wildcard Let's Encrypt cert per domain, renewed by certbot's **DNS-01 challenge** through the DNS provider's API. DNS-01 is required for wildcards and has the side benefit that no HTTP challenge port needs to be open during renewal.

## 2. IP addressing — who lives where

```
Internet (public IPv4, dynamic)        Internet (public IPv4, static)
        │                                       │
   consumer router                              │
   192.168.4.0/22 LAN                           │
        │                                       │
   home server ─────── Tailscale ────────────── VPS
   enp1s0    192.168.4.x/22   (WireGuard, CGNAT 100.64.0.0/10)   eth0  static
   tailscale0 100.x.y.z/32                                        tailscale0 100.a.b.c/32
   docker0    172.17.0.1/16
   plex_network 172.20.0.0/16  ← all 51 containers
```

- **LAN `192.168.4.0/22`** — the home network. Only the home server's own interface matters here; nothing else on the LAN is part of the stack.
- **Docker bridge `172.20.0.0/16`** — every container gets an address here. Containers talk to each other on this network; the host publishes selected ports to the outside.
- **Tailnet `100.64.0.0/10`** — Tailscale hands each node one stable `/32` from the CGNAT range. It never changes, it works from any physical network, and it is the address services bind to when they should be reachable by the fleet but not the internet.
- **Public IPs** — only nginx (80/443) and Plex (32400) are ever meant to be reached on them.

## 3. Firewall — UFW on the host, Tailscale's chain in front of it

Default policy on both servers: **deny incoming, allow outgoing.** The rules that matter on the home server:

| Rule | Purpose |
|---|---|
| `80/tcp`, `443/tcp` from anywhere | nginx reverse proxy (all web UIs, behind OAuth2) |
| `32400/tcp` from anywhere | Plex direct connections |
| `22` from `172.18/16`, `172.20/16` | Docker networks → host SSH, so the WeTTY web terminal can reach the host |
| `22` from one peer IP | The VPS, as a break-glass path if the tailnet is down |
| `3389 on tailscale0` | RDP, tailnet only |
| `11235/tcp on tailscale0` | Crawl4AI for the VPS's agents, tailnet only |
| one high port tcp+udp | Torrent client port-forward |

**There is no public SSH rule.** Fleet SSH arrives on `tailscale0`, and Tailscale installs its own netfilter chain (`ts-input`) ahead of UFW that accepts traffic on that interface and drops anything claiming a `100.64/10` source on any *other* interface. So SSH is open to the tailnet and closed to the internet, without touching sshd's listen address. On the VPS the same pattern holds with a shorter list: UFW allows only nginx, and SSH is tailnet-only.

Behind UFW: **fail2ban** jails on sshd and nginx auth failures, and OAuth2 in front of every web UI so a reachable port is not the same as a reachable application.

## 4. Routing and traffic paths

**Public web UI request**
```
browser → DNS (wildcard A) → :443 nginx
        → auth_request → oauth2-proxy (Google, allow-listed emails)
        → proxy_pass → container's published port on 127.0.0.1
```
Every subdomain is one nginx vhost with the same TLS snippet and the same auth block. Adding a service is a DNS-free operation because the wildcard already covers it.

**Fleet-internal request**
```
VPS process → 100.x.y.z:11235  →  WireGuard tunnel  →  home server tailscale0  →  Crawl4AI container
```
No DNS record, no TLS termination, no OAuth: the tunnel is authenticated and encrypted end to end, and the port is bound to the tailnet address so it does not exist on the public interface.

**Torrent client**
```
qbittorrent-vpn container → OpenVPN tunnel → provider exit
                          └─ kill switch: if the tunnel drops, all traffic stops
```
The client's traffic never uses the home connection directly. A separate leak-detection container confirms the exit IP is the VPN's.

**Egress through the home IP from elsewhere**
```
laptop / VPS → SOCKS5 proxy on home server's tailnet address :1080 → home residential IP → target site
```
Used for services that only behave for a residential IP. The proxy is bound to the tailnet address, so it is a fleet-only exit, not an open proxy.

## 5. Tailnet specifics

| Item | Setting | Why |
|---|---|---|
| MagicDNS | on | Short names everywhere; no `/etc/hosts` maintenance |
| Key expiry | **disabled** on the three always-on nodes; default 180-day on laptops and phone | Servers must not silently fall off the network; portable devices should re-auth periodically |
| Exit nodes / subnet routers | not used (yet) | Nothing on the LAN needs exposing beyond the server itself; a subnet router would be the next step if it did |
| ACLs | default (all nodes see all nodes) | Single-user tailnet; the interesting boundary is which *ports* bind to the tailnet address, handled per service |
| Monitoring | [tailscale-fleet-watchdog](https://github.com/tylerbcrawford/tailscale-fleet-watchdog) | Daily per-node self-check, weekly REST-API fleet audit, Discord alerts on transitions only |

## 6. Things that bit me

- **A node key expired and nothing said so.** The daemon kept running, the control plane rejected the node, and the first symptom was `ssh: connection timed out` from another machine. Fix: disable expiry on always-on nodes and run a watchdog that reads `tailscale status --json` and the `/devices` API.
- **Wildcard DNS makes dead subdomains return 200.** Any typo resolves to nginx, which happily serves the default vhost. Health checks must check the page title or a known string, not the status code.
- **A residential IP without dynamic DNS is a service outage waiting to happen.** The 5-minute updater exists because a lease change once took every public subdomain down until it was noticed.
- **Docker-published ports bypass UFW.** A `-p 8080:80` publishes on `0.0.0.0` via Docker's own iptables rules, regardless of UFW. Anything private is published on `127.0.0.1:` or on the tailnet address explicitly.
