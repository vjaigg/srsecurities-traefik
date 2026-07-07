# SR Securities — Traefik Edge Proxy

Centralized edge reverse proxy for SR Securities infrastructure.  
One Traefik v2.11 instance handles all TLS termination, routing, and certificate management.

**Author:** [imvijaychaurasia](https://github.com/imvijaychaurasia)

---

## What this does

- Terminates HTTPS using **Let's Encrypt wildcard certificates** (DNS-01 via GoDaddy API)
- Currently serves: `timenexahr.com` + `*.timenexahr.com`
- Forces HTTP → HTTPS globally
- Adds security headers to all responses
- Proxies traffic to internal services on the LAN
- Exposes the Traefik dashboard at `https://traefik.timenexahr.com/dashboard/`

---

## Infrastructure

```
Public DNS → 111.125.252.176 (PFSense NAT) → 172.22.66.25 (Traefik VM)
                                                      ↓
                                          172.22.66.21:8000  (TimeNexaHR / SRS VM)
                                          172.22.66.10:80    (essl.in — pending)
```

| Host | LAN IP | Role |
|---|---|---|
| Traefik VM | `172.22.66.25` | This edge proxy |
| SRS VM (TimeNexaHR) | `172.22.66.21` | Django app server (Gunicorn :8000) |
| essl.in server | `172.22.66.10` | Pending (essl.in domain key needed) |

**PFSense NAT rules required** (Firewall → NAT → Port Forward):

| WAN Port | → Internal IP | → Port |
|---|---|---|
| 80 | 172.22.66.25 | 80 |
| 443 | 172.22.66.25 | 443 |

> Port 4000 (dashboard) should NOT be forwarded to public — LAN access only via `https://traefik.timenexahr.com:4000/` or direct `http://172.22.66.25:4000/`.

---

## File layout

```
srsecurities-traefik/
├── docker-compose.yml          # Traefik service definition
├── .env                        # Secrets — NOT committed (see .env.example)
├── letsencrypt/
│   └── acme.json               # chmod 600 — LE cert storage (auto-managed)
└── dynamic/                    # Hot-reloaded config (Traefik watches this dir)
    ├── tls-and-headers.yaml    # TLS options + shared middlewares
    ├── cert-preload.yaml       # Pre-warms wildcard certs on startup
    ├── timenexahr.com.yaml     # Routes for timenexahr.com services
    ├── dashboard.yaml          # Traefik dashboard routes
    └── essl.in.yaml.disabled   # Pending (rename to .yaml when key is ready)
```

> **Important:** Traefik only reads `.yaml` / `.yml` files from `dynamic/`. Files with any other extension (e.g. `.yaml.disabled`) are silently ignored.

---

## Prerequisites

- Docker + Docker Compose installed on the Traefik VM (172.22.66.25)
- GoDaddy API key/secret for DNS-01 challenge (covers `timenexahr.com`)
- DNS A records pointing to `111.125.252.176` for all domains you want to serve

---

## First-time setup

```bash
# 1. Clone this repo on the Traefik VM
git clone git@github.com:vjaigg/srsecurities-traefik.git ~/apps/srsecurities-traefik
cd ~/apps/srsecurities-traefik

# 2. Create secrets file
cp .env.example .env
nano .env   # fill in values (see .env.example)

# 3. Create and lock down ACME storage
mkdir -p letsencrypt
touch letsencrypt/acme.json && chmod 600 letsencrypt/acme.json

# 4. Start Traefik
docker compose up -d

# 5. Check logs (cert issuance takes ~60s on first run)
docker logs -f edge-traefik
```

### `.env` values

```env
LE_EMAIL=hello@timenexahr.com
GODADDY_API_KEY=<your-godaddy-key>
GODADDY_API_SECRET=<your-godaddy-secret>
```

---

## Add a route for a new service

Create a new file in `dynamic/` — Traefik hot-reloads it within seconds, no restart needed.

**Template: `dynamic/myservice.yaml`**

```yaml
http:
  routers:
    myservice:
      rule: "Host(`myservice.timenexahr.com`)"
      entryPoints: ["websecure"]
      service: myservice-svc
      middlewares: ["security-headers", "compress"]
      tls:
        certResolver: le          # uses existing *.timenexahr.com wildcard

  services:
    myservice-svc:
      loadBalancer:
        servers:
          - url: "http://172.22.66.XX:PORT"
        passHostHeader: true
```

**Rules:**
- Use `certResolver: le` — Traefik automatically picks the matching wildcard cert from `acme.json`
- `passHostHeader: true` — backend receives the original `Host:` header
- Always add `middlewares: ["security-headers"]` at minimum

---

## Add a wildcard certificate for a new domain

When you add a brand-new domain (e.g. `newdomain.com`) not covered by existing certs, add a preload entry to `dynamic/cert-preload.yaml`:

```yaml
http:
  routers:
    cert-preload-newdomain:
      rule: "Host(`newdomain.com`)"
      entryPoints: ["websecure"]
      service: noop@internal
      tls:
        certResolver: le
        domains:
          - main: "newdomain.com"
            sans: ["*.newdomain.com"]
```

**Requirements before adding:**
1. DNS A record for `newdomain.com` must point to `111.125.252.176`
2. The GoDaddy API key must cover `newdomain.com` (same account = same key)
3. If the domain is on a **different GoDaddy account**, you need a separate cert resolver with that account's key

**For a domain on a different DNS provider / GoDaddy account:**

Add a new resolver in `docker-compose.yml`:

```yaml
- "--certificatesresolvers.le2.acme.email=${LE_EMAIL}"
- "--certificatesresolvers.le2.acme.storage=/letsencrypt/acme.json"
- "--certificatesresolvers.le2.acme.dnschallenge=true"
- "--certificatesresolvers.le2.acme.dnschallenge.provider=godaddy"
- "--certificatesresolvers.le2.acme.dnschallenge.delaybeforecheck=30"
- "--certificatesresolvers.le2.acme.dnschallenge.resolvers=1.1.1.1:53,8.8.8.8:53"
```

Add second key pair to `.env`:
```env
GODADDY2_API_KEY=<second-account-key>
GODADDY2_API_SECRET=<second-account-secret>
```

Then reference `certResolver: le2` in your route.

> **essl.in** is currently disabled (`essl.in.yaml.disabled`) because it requires a GoDaddy key from the essl.in domain owner's account. Rename to `essl.in.yaml` and add `GODADDY2_*` env vars when that key is available.

---

## Operate

### Check Traefik status

```bash
# Container status
docker ps

# Live logs
docker logs -f edge-traefik

# ACME / cert activity only
docker logs edge-traefik 2>&1 | grep -i "acme\|certificate\|challenge\|level=error"
```

### Verify a certificate

```bash
# Check cert served for a domain (with SNI)
echo | openssl s_client -connect 172.22.66.25:443 -servername demo.timenexahr.com 2>&1 | grep -E "subject|issuer|CN="

# Full curl with SNI (bypass DNS, test directly against Traefik)
curl -sk --resolve demo.timenexahr.com:443:172.22.66.25 https://demo.timenexahr.com -I
```

### Verify a route is loaded

```bash
# Check active routers via Traefik API (from inside LAN)
curl -s http://172.22.66.25:4000/api/http/routers | python3 -m json.tool | grep "name\|rule\|status"
```

### Access the dashboard

| Method | URL | Requirement |
|---|---|---|
| Via domain (HTTPS) | `https://traefik.timenexahr.com/dashboard/` | DNS + PFSense NAT port 443 forwarded |
| Via LAN IP | `http://172.22.66.25:4000/dashboard/` | Must be on 172.22.66.x LAN |
| Via domain + port | `https://traefik.timenexahr.com:4000/dashboard/` | LAN only |

### Reload config

No restart needed. Just save a `.yaml` file in `dynamic/` — Traefik picks it up in ~1 second (`--providers.file.watch=true`).

To force restart:

```bash
docker compose restart
# or full recreate:
docker compose down && docker compose up -d
```

### Backup / restore ACME certs

```bash
# Backup (do this before any changes)
cp letsencrypt/acme.json letsencrypt/acme.json.backup-$(date +%F)

# Restore
cp letsencrypt/acme.json.backup-YYYY-MM-DD letsencrypt/acme.json
chmod 600 letsencrypt/acme.json
docker compose restart
```

> ⚠️ If you clear `acme.json` (or it's corrupt), Traefik re-issues all certs from scratch. Let's Encrypt rate limits: 5 duplicate certs per domain per week.

---

## Troubleshooting

### `curl: (60) SSL certificate problem: self-signed certificate`

Traefik is serving its fallback self-signed cert. Causes:
- No SNI in the request (you used an IP directly) — expected, use `--resolve` or `--insecure` to test
- The ACME cert hasn't been issued yet — check logs for `Server responded with a certificate`
- The route's `certResolver: le` is missing

### `Forbidden` on dashboard

The `allow-lan-only` middleware (if still active) is blocking your IP. Access via `http://172.22.66.25:4000/dashboard/` from the LAN.

### `Connection timed out` on port 443

PFSense NAT port forward for 443 → 172.22.66.25 is missing or disabled. Add it under Firewall → NAT → Port Forward.

### `Unable to obtain ACME certificate: UNKNOWN_DOMAIN`

The GoDaddy API key doesn't manage that domain. Either:
- The domain is on a different GoDaddy account → use a separate cert resolver with the correct key
- The domain isn't registered in GoDaddy at all → use the correct DNS provider

### ACME cert not issuing / DNS challenge failing

```bash
# Watch ACME activity
docker logs -f edge-traefik 2>&1 | grep -i "dns01\|challenge\|godaddy\|error"
```

Common causes:
- GoDaddy key expired or wrong — verify at developer.godaddy.com
- DNS not propagated yet — `delaybeforecheck=30` gives 30s, increase if flaky
- Route defined in a subdirectory of `dynamic/` — Traefik only reads top-level `.yaml` files (no subdirs)

### Backend returning 502 Bad Gateway

The backend is unreachable from Traefik:
```bash
# Test from Traefik VM
curl -v http://172.22.66.21:8000/

# If refused: backend is bound to 127.0.0.1 only
# Fix: bind to 0.0.0.0 or the host LAN IP
```

For Django/Gunicorn: check docker-compose port binding (`172.22.66.21:8000:8000`, not `127.0.0.1:8000:8000`).

### Check which cert is being served at the public IP

```bash
echo | openssl s_client -connect 111.125.252.176:443 -servername demo.timenexahr.com 2>&1 | grep "CN="
```

If you see a wrong cert (e.g. PFSense's own cert), a HAProxy frontend or the PFSense web GUI is intercepting port 443 before Traefik. Disable it in PFSense and ensure the NAT rule forwards correctly.

---

## Currently active routes

| Domain | Backend | Notes |
|---|---|---|
| `demo.timenexahr.com` | `172.22.66.21:8000` | TimeNexaHR Django app |
| `traefik.timenexahr.com` | `api@internal` | Traefik dashboard |
| `online.essl.in` | `172.22.66.10:80` | **Disabled** — needs essl.in GoDaddy key |

---

## Security notes

- `acme.json` contains private keys — **never commit it** (already in `.gitignore`)
- `.env` contains GoDaddy API secrets — **never commit it** (already in `.gitignore`)
- The dashboard has no authentication — keep port 4000 LAN-only (do not forward in PFSense)
- `security-headers` middleware adds HSTS, X-Frame-Options, and CSP headers to all responses
