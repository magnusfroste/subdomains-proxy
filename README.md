# 🪄 DomainProxy

**Custom domains for your SaaS — Cloud or Self-Hosted**

Give your customers branded subdomains with automatic HTTPS. Like Cloudflare for SaaS, but free and open source.

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Docker](https://img.shields.io/badge/Docker-Ready-blue.svg)](docker-compose.yml)

## 🚀 Choose Your Deployment

| Option | Best For | Get Started |
|--------|----------|-------------|
| ☁️ **Cloud Proxy** | No infrastructure needed | [subdomains.site](https://subdomains.site) |
| 🏠 **Self-Hosted** | Full control, own data | See below |

## ☁️ Cloud Proxy (Recommended)

Use the hosted service — no servers, no certificates, no maintenance:

1. Go to [subdomains.site/admin](https://subdomains.site/admin)
2. Create a SaaS account to get your API key
3. Start registering subdomains via API

## How It Works

1. **You register a subdomain** via API: `career.customer.com → your-app.com`
2. **Customer sets DNS**: `career.customer.com CNAME subdomains.site`
3. **Automatic HTTPS**: Certificate issued on first visit via Let's Encrypt
4. **Proxy forwards traffic**: Requests go to your app with original host header

```
Customer visits: https://career.customer.com
        ↓
   DomainProxy (TLS termination + cert)
        ↓
   Your SaaS app (receives X-Forwarded-Host: career.customer.com)
```

## API Quick Start

```bash
# 1. Create tenant (customer's base domain)
curl -X POST https://subdomains.site/api/v1/create-tenant \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"base_domain": "customer.com"}'

# 2. Register subdomain → your app
curl -X POST https://subdomains.site/api/v1/register-subdomain \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"subdomain": "career", "base_domain": "customer.com", "target_url": "https://your-app.com"}'

# 3. Customer adds DNS: career.customer.com CNAME subdomains.site
# 4. Visit https://career.customer.com — HTTPS just works! 🎉
```

> **Self-hosted?** Replace `subdomains.site` with your own domain in all API calls.

## API Reference

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/create-tenant` | POST | Create a tenant (base domain) |
| `/api/v1/register-subdomain` | POST | Register subdomain proxy |
| `/api/v1/delete-proxy` | POST | Delete subdomain proxy |
| `/api/v1/tenants` | GET | List your tenants |
| `/api/v1/proxies` | GET | List your proxies |
| `/api/v1/status` | GET | Health check |
| `/api/v1/verify-domain` | GET | Check if domain is registered (used by Caddy internally) |
| `/api/v1/integration-guide` | GET | Full integration guide (Markdown) |

Full documentation: [subdomains.site/docs](https://subdomains.site/docs)

---

## 🏠 Self-Hosted Production Deployment

### Prerequisites

- VPS with ports 80 and 443 open (any provider — DigitalOcean, Hetzner, etc.)
- A domain with wildcard DNS pointing to your server IP:
  ```
  yourdomain.com    A  →  your-server-ip
  *.yourdomain.com  A  →  your-server-ip
  ```
- Docker installed:
  ```bash
  curl -fsSL https://get.docker.com | sh
  ```

### Deploy

**1. Clone and configure**
```bash
git clone https://github.com/magnusfroste/domainproxy.git
cd domainproxy
cp .env.example .env
```

**2. Edit `.env`**
```bash
nano .env
```

| Variable | Description | Default |
|----------|-------------|---------|
| `ADMIN_PASS` | Admin password — **change this!** | `changeme` |
| `CADDY_EMAIL` | Email for Let's Encrypt (required) | — |

**3. Edit `Caddyfile`**

The `Caddyfile` lives in the root of the repo. Docker mounts it into the Caddy container automatically — you do not install Caddy separately.

Open it and replace `yourdomain.com` with your actual domain:
```bash
nano Caddyfile
```

The file should look like this (only change the domain line):
```
{
  email {$CADDY_EMAIL}
  on_demand_tls {
    ask http://proxy:3000/api/v1/verify-domain
  }
}

yourdomain.com {
  reverse_proxy proxy:3000
}

https:// {
  tls { on_demand }
  reverse_proxy proxy:3000
}

http:// {
  reverse_proxy proxy:3000
}
```

> **How TLS works:** Caddy runs as a Docker container and handles all HTTPS automatically. When a customer visits `career.acme.com` for the first time, Caddy asks DomainProxy `/api/v1/verify-domain?domain=career.acme.com`. If the domain is registered, Caddy fetches a Let's Encrypt certificate on the spot — no manual steps needed. Certificates are cached in a Docker volume and renewed automatically.

**4. Start**
```bash
docker compose -f docker-compose.prod.yml up -d
```

Open `https://yourdomain.com/admin` to create your first SaaS account.

> ⚠️ Make sure DNS has propagated before starting — Let's Encrypt needs to reach your server on port 80 to issue the first certificate for your admin domain.

### Security

> ⚠️ **Change the default admin password** before exposing this to the internet. Set `ADMIN_PASS` in `.env`.

---

## Tech Stack

- **Runtime:** Node.js + Express
- **Database:** SQLite (zero config, no separate DB server)
- **TLS:** Caddy with Let's Encrypt on-demand certificates
- **Container:** Docker + Docker Compose

## Contributing

PRs welcome! Please open an issue first to discuss major changes.

## License

MIT — do whatever you want.

---

**Cloud:** [subdomains.site](https://subdomains.site) · **Docs:** [subdomains.site/docs](https://subdomains.site/docs) · **GitHub:** [magnusfroste/domainproxy](https://github.com/magnusfroste/domainproxy)
