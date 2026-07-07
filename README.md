# SR Securities — Traefik Edge Proxy

Centralized edge reverse proxy for SR Securities infrastructure.  
One Traefik v2.11 instance handles all TLS termination, routing, and certificate management.

**Author:** [imvijaychaurasia](https://github.com/imvijaychaurasia)

---

## What this does

- Terminates HTTPS using **Let's Encrypt wildcard certificates** (DNS-01 via GoDaddy API)
- Currently serves:
  - `timenexahr.com` + `*.timenexahr.com` — TimeNexaHR HRMS platform
  - `onlineessl.in` + `*.onlineessl.in` — 326 tenant subdomains (ports 50000–50325)
- Forces HTTP → HTTPS globally
- Adds security headers to all responses
- Proxies traffic to internal services on the LAN
- Exposes the Traefik dashboard at `https://traefik.timenexahr.com/dashboard/`

---

## Infrastructure

```
Public DNS → 111.125.252.176 (PFSense NAT) → 172.22.66.25 (Traefik VM)
                                                      ↓
                                          172.22.66.21:8000   (TimeNexaHR / SRS VM)
                                          172.22.66.10:50000+ (onlineessl.in tenants)
```

| Host | LAN IP | Role |
|---|---|---|
| Traefik VM | `172.22.66.25` | This edge proxy |
| SRS VM (TimeNexaHR) | `172.22.66.21` | Django app server (Gunicorn :8000) |
| onlineessl.in server | `172.22.66.10` | 326 tenant apps on ports 50000–50325 |

**PFSense NAT rules required** (Firewall → NAT → Port Forward):

| WAN Port | → Internal IP | → Port |
|---|---|---|
| 80 | 172.22.66.25 | 80 |
| 443 | 172.22.66.25 | 443 |

> Port 4000 (dashboard) should NOT be forwarded to public — accessible at `https://traefik.timenexahr.com/dashboard/` (port 443) or directly via `http://172.22.66.25:4000/dashboard/` on LAN.

---

## File layout

```
srsecurities-traefik/
├── docker-compose.yml           # Traefik service definition
├── .env                         # Secrets — NOT committed (see .env.example)
├── .env.example                 # Template for .env
├── letsencrypt/
│   └── acme.json                # chmod 600 — LE cert storage (auto-managed)
└── dynamic/                     # Hot-reloaded config (Traefik watches this dir)
    ├── tls-and-headers.yaml     # TLS options + shared middlewares
    ├── cert-preload.yaml        # Pre-warms wildcard certs on startup
    ├── timenexahr.com.yaml      # Routes for timenexahr.com services
    ├── onlineessl.in.yaml       # 326 tenant routes for onlineessl.in (auto-generated)
    ├── dashboard.yaml           # Traefik dashboard routes
    └── essl.in.yaml.disabled    # Pending (rename to .yaml when GoDaddy key available)
```

> **Important:** Traefik only reads `.yaml` / `.yml` files from `dynamic/`. Files with any other extension (e.g. `.yaml.disabled`) are silently ignored — use this to disable a config without deleting it.

---

## Currently active routes

### timenexahr.com

| Domain | Backend | Status |
|---|---|---|
| `demo.timenexahr.com` | `172.22.66.21:8000` | ✅ Live |
| `traefik.timenexahr.com` | `api@internal` | ✅ Dashboard |

### onlineessl.in (326 tenant subdomains)

All subdomains follow the pattern `<slug>.onlineessl.in → 172.22.66.10:<port>`.

| Port range | Count | Status |
|---|---|---|
| 50000–50325 | 326 routes | ✅ 319 live, 7 backend down |

**7 backends currently down** (Traefik routing fine — containers not running on 172.22.66.10):

| Subdomain | Port |
|---|---|
| `dbt.onlineessl.in` | 50002 |
| `nxtspire.onlineessl.in` | 50004 |
| `rmrssachiv.onlineessl.in` | 50017 |
| `vidyasankalp.onlineessl.in` | 50035 |
| `bbl.onlineessl.in` | 50074 |
| `rkd.onlineessl.in` | 50141 |
| `eonmatrix.onlineessl.in` | 50202 |

> To fix a 502: start the corresponding container on `172.22.66.10` — no Traefik changes needed.

### Disabled / pending

| Domain | File | Reason |
|---|---|---|
| `essl.in` / `*.essl.in` | `essl.in.yaml.disabled` | GoDaddy key for essl.in not yet available |

---

## Wildcard certificates

| Domain | Resolver | Status |
|---|---|---|
| `timenexahr.com` + `*.timenexahr.com` | `le` (GoDaddy) | ✅ Issued, auto-renewing |
| `onlineessl.in` + `*.onlineessl.in` | `le` (GoDaddy) | ✅ Issued, auto-renewing |
| `traefik.timenexahr.com` | `le` (GoDaddy) | ✅ Issued, auto-renewing |

Both `timenexahr.com` and `onlineessl.in` are managed by the **same GoDaddy account** — one API key covers both.

---

## Prerequisites

- Docker + Docker Compose installed on the Traefik VM (172.22.66.25)
- GoDaddy API key/secret for DNS-01 challenge (covers `timenexahr.com` and `onlineessl.in`)
- DNS A records pointing to `111.125.252.176` for all domains you want to serve

---

## First-time setup

```bash
# 1. Clone this repo on the Traefik VM
git clone git@github.com:vjaigg/srsecurities-traefik.git ~/apps/srsecurities-traefik
cd ~/apps/srsecurities-traefik

# 2. Create secrets file
cp .env.example .env
nano .env   # fill in values

# 3. Create and lock down ACME storage
mkdir -p letsencrypt
touch letsencrypt/acme.json && chmod 600 letsencrypt/acme.json

# 4. Start Traefik
docker compose up -d

# 5. Check logs (cert issuance takes ~60–90s on first run)
docker logs -f edge-traefik
```

### `.env` values

```env
LE_EMAIL=hello@timenexahr.com
GODADDY_API_KEY=<your-godaddy-api-key>
GODADDY_API_SECRET=<your-godaddy-api-secret>
```

---

## Add a route for a new `timenexahr.com` service

Create a new file in `dynamic/` — Traefik hot-reloads within ~1 second, no restart needed.

```yaml
# dynamic/myservice.yaml
http:
  routers:
    myservice:
      rule: "Host(`myservice.timenexahr.com`)"
      entryPoints: ["websecure"]
      service: myservice-svc
      middlewares: ["security-headers", "compress"]
      tls:
        certResolver: le    # picks up existing *.timenexahr.com wildcard automatically

  services:
    myservice-svc:
      loadBalancer:
        servers:
          - url: "http://172.22.66.XX:PORT"
        passHostHeader: true
```

---

## Add a new `onlineessl.in` tenant subdomain

Append to `dynamic/onlineessl.in.yaml` — both a router and a service entry are needed:

```yaml
# Under http.routers:
    newslug:
      rule: "Host(`newslug.onlineessl.in`)"
      entryPoints: ["websecure"]
      service: newslug-svc
      middlewares: ["security-headers", "compress"]
      tls:
        certResolver: le    # picks up existing *.onlineessl.in wildcard

# Under http.services:
    newslug-svc:
      loadBalancer:
        servers:
          - url: "http://172.22.66.10:5XXXX"
        passHostHeader: true
```

The `*.onlineessl.in` wildcard cert covers all subdomains — no cert changes needed for new tenants.

---

## Add a wildcard certificate for a brand-new domain

When adding a domain not yet covered (e.g. `newdomain.com`), add a preload entry to `dynamic/cert-preload.yaml`:

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
3. If domain is on a **different GoDaddy account**, add a second resolver (see below)

**For a domain on a different GoDaddy account:**

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

Then use `certResolver: le2` in routes for that domain.

> `essl.in` is currently disabled (`essl.in.yaml.disabled`) — it's on a different GoDaddy account. Rename to `essl.in.yaml` and add `GODADDY2_*` env vars when the essl.in account key is provided.

---

## Operate

### Check Traefik status

```bash
docker ps                           # container health
docker logs -f edge-traefik         # live logs
docker logs edge-traefik 2>&1 | grep -i "error\|acme\|certificate"   # cert activity
```

### Verify a certificate

```bash
# Check cert served (with SNI) — from Traefik VM
echo | openssl s_client -connect 172.22.66.25:443 -servername demo.onlineessl.in 2>&1 | grep "CN="

# Full end-to-end test bypassing DNS
curl -sk --resolve demo.onlineessl.in:443:172.22.66.25 https://demo.onlineessl.in -I
```

### Batch test all onlineessl.in subdomains

```bash
# Quick status check — shows which backends are up (200) vs down (502)
while IFS=',' read sub port; do
  code=$(curl -sk --max-time 5 -o /dev/null -w "%{http_code}" https://$sub/)
  echo "$sub (port $port) → $code"
done < <(grep "url:" dynamic/onlineessl.in.yaml | awk -F: '{print $NF}')
```

### Access the dashboard

| Method | URL | Requirement |
|---|---|---|
| Via domain | `https://traefik.timenexahr.com/dashboard/` | Public (NAT port 443 forwarded) |
| Via LAN IP | `http://172.22.66.25:4000/dashboard/` | LAN only |
| Via domain + port | `https://traefik.timenexahr.com:4000/dashboard/` | LAN only |

### Reload config

```bash
# No restart needed — save any .yaml in dynamic/ and Traefik picks it up instantly

# Force restart if needed
docker compose restart

# Full recreate
docker compose down && docker compose up -d
```

### Backup ACME certs

```bash
cp letsencrypt/acme.json letsencrypt/acme.json.backup-$(date +%F)
```

> ⚠️ Let's Encrypt rate limit: **5 duplicate certs per domain per week**. Always backup before clearing `acme.json`.

---

## Troubleshooting

### Self-signed cert (`TRAEFIK DEFAULT CERT`)

Traefik falls back to its self-signed cert when:
- Request uses a bare IP (no SNI) — expected, use `--resolve` flag to test with SNI
- ACME cert not yet issued — check logs for `Server responded with a certificate`
- Route missing `certResolver: le`

### `Forbidden` on dashboard

Access the dashboard from the LAN (`http://172.22.66.25:4000/dashboard/`) or via `https://traefik.timenexahr.com/dashboard/` (port 443, publicly reachable).

### `Connection timed out` on port 443

PFSense NAT port forward for `443 → 172.22.66.25` is missing or disabled. Check Firewall → NAT → Port Forward.

### `Unable to obtain ACME certificate: UNKNOWN_DOMAIN`

The GoDaddy API key doesn't manage that domain:
- Domain is on a different GoDaddy account → add a second cert resolver with the correct key
- Domain isn't in GoDaddy at all → use the matching DNS provider

### ACME / DNS challenge failing

```bash
docker logs -f edge-traefik 2>&1 | grep -i "dns01\|challenge\|godaddy\|error"
```

Common causes:
- GoDaddy key expired → verify at [developer.godaddy.com](https://developer.godaddy.com)
- DNS record not propagated → increase `delaybeforecheck` in `docker-compose.yml`
- Config file in a subdirectory of `dynamic/` → Traefik only reads **top-level** `.yaml` files

### 502 Bad Gateway

Backend unreachable from Traefik:
```bash
# Test from Traefik VM directly
curl -v http://172.22.66.10:50002/    # example: dbt.onlineessl.in

# If refused: container not running on 172.22.66.10
# If timeout: firewall blocking Traefik → 172.22.66.10
```

### Wrong cert at public IP (e.g. PFSense cert)

```bash
echo | openssl s_client -connect 111.125.252.176:443 -servername demo.timenexahr.com 2>&1 | grep "CN="
```

If you see a PFSense or HAProxy cert: a frontend is intercepting port 443 before Traefik. Disable it in PFSense (Services → HAProxy → Frontend) and verify the NAT Port Forward rule is in place.

---

## Security notes

- `acme.json` contains private TLS keys — **never commit it** (in `.gitignore`)
- `.env` contains GoDaddy API secrets — **never commit it** (in `.gitignore`)
- The dashboard has no login/password — keep it behind the LAN or add BasicAuth middleware
- `security-headers` middleware enforces HSTS, X-Frame-Options, XSS protection on all responses
