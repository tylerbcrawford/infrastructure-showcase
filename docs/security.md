# Security Hardening

## Authentication: OAuth2 Proxy

### How It Works

Every web-facing service is protected by a centralized [OAuth2 Proxy](https://oauth2-proxy.github.io/oauth2-proxy/) instance using the **Google provider**.

```
User --> nginx --> auth_request to OAuth2 Proxy
                       |
                       +--> Not authenticated? Redirect to Google login
                       |
                       +--> Authenticated? Proxy to upstream service
```

### Configuration Highlights

| Setting | Value | Why |
|---------|-------|-----|
| Provider | Google OAuth2 | Reliable, supports email filtering |
| Cookie Expiry | 7 days (168h) | Balance between convenience and security |
| Cookie Refresh | 24 hours | Re-validate Google session daily |
| Cookie Domain | `.yourdomain.com` | Single sign-on across all subdomains |
| Session Store | Cookie-based | No Redis/DB dependency, simpler architecture |
| Email Filtering | Allowlist file | Only specific Google accounts can access |
| Skip Provider Button | Yes | Direct redirect to Google (no intermediate page) |

### Email Allowlist

OAuth2 Proxy uses an authenticated emails file (`.config/.oauth2-emails.txt`) to restrict access. Only listed email addresses can authenticate, even though `EMAIL_DOMAINS=*` is set.

### nginx Integration

Each service's nginx vhost includes:

1. **`/oauth2/auth` location**: Proxies auth check to OAuth2 Proxy
2. **`@oauth2_signin` error handler**: Redirects to `auth.yourdomain.com` on 401/403
3. **Auth headers forwarded**: Email, username, and JWT passed to upstream services

### Services Without OAuth2

Some services intentionally bypass OAuth2:

| Service | Auth Method | Reason |
|---------|-------------|--------|
| Kobo Sync | Token in URL path | Kobo devices can't do OAuth2 |
| Twilio SMS | HMAC-SHA1 signature | Twilio webhook verification |
| Plex | Built-in auth | Plex has its own account system |
| Mousehole | Internal only | VPN leak detection, not exposed externally |

## SSL / TLS

### Wildcard Certificate

A single wildcard certificate from Let's Encrypt covers `*.yourdomain.com`:

```bash
# Renewal via certbot with DNS plugin
certbot certonly --dns-<provider> -d "*.yourdomain.com" -d "yourdomain.com"
```

### nginx SSL Config

All vhosts include a shared SSL snippet:

```nginx
include /etc/nginx/snippets/ssl-yourdomain.com.conf;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
```

- **HSTS**: Enforced with preload
- **HTTP to HTTPS redirect**: All port 80 traffic redirected to 443

## Firewall (UFW)

```bash
# Default policy
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allowed ports
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP (redirects to HTTPS)
sudo ufw allow 443/tcp   # HTTPS
sudo ufw allow 32400/tcp # Plex (direct connections)
```

All other ports (8989, 7878, 6789, etc.) are only accessible through the nginx reverse proxy on port 443.

## fail2ban

### SSH Protection

```ini
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
findtime = 600       # 10 minutes
bantime = 3600       # 1 hour
```

### nginx Protection

```ini
[nginx-http-auth]
enabled = true
filter = nginx-http-auth
logpath = /var/log/nginx/error.log
maxretry = 5
findtime = 600
bantime = 3600
```

## SSH Hardening

| Setting | Value |
|---------|-------|
| PasswordAuthentication | no (key-only) |
| PermitRootLogin | no |
| X11Forwarding | no |
| MaxAuthTries | 3 |

## VPN for Download Clients

The primary torrent client (`qbittorrent-vpn`) runs inside a VPN container:

- **VPN Provider**: Custom OpenVPN configuration
- **Kill Switch**: `STRICT_PORT_FORWARD=yes` -- if VPN drops, all traffic stops
- **Verification**: Health check confirms external IP matches VPN endpoint

```bash
# Verify VPN is active (should show VPN IP, not local)
docker exec qbittorrent-vpn curl -s checkip.amazonaws.com
```

## Docker Security

### Resource Limits

Every container has CPU and memory limits via `deploy.resources.limits`. No container can consume unlimited host resources.

### Minimal Privileges

- **No `privileged: true`**: Glances uses targeted `cap_add: [SYS_PTRACE]` instead
- **Read-only mounts**: Media files mounted as `:ro` where write access isn't needed
- **Docker socket**: Only Portainer, Glances, WUD, and Dashboard have socket access (required for their functionality). WUD's mount is read-only (`:ro`).

### Health Checks

18 containers have Docker health checks (HTTP probes for web UIs, process checks for daemons). This enables:

- Automatic restart on failure (`restart: on-failure`)
- Dependency ordering (`depends_on: condition: service_healthy`)
- Monitoring visibility via `docker compose ps`

### Container Log Limits

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

Prevents any single container from filling the disk with logs. Maximum 30MB per container (3 x 10MB rotated files).

## Kernel Hardening (sysctl)

```ini
# /etc/sysctl.d/99-zz-hardening.conf
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.log_martians = 1
```

## Unattended Security Updates

```bash
# Ubuntu automatic security patches
sudo dpkg-reconfigure -plow unattended-upgrades
```

Critical security patches are applied automatically. Non-security updates are manual.
