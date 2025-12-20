# Architecture

This document describes the architecture and installation scenarios supported by the remnawave-installer.

## Installation Scenarios

### Option 1: Two Servers (Panel and Node separately)

Recommended for better reliability and flexibility.

**Panel Server:**
- Remnawave Panel
- Subscription-Page
- Caddy (for panel and subscriptions)
- PostgreSQL, Redis

**Node Server:**
- Remnawave Node (with Xray)
- Caddy for Selfsteal domain

**DNS Configuration:**
- Panel and Subscription domains → Panel server IP
- Selfsteal domain → Node server IP

**Installation Order:**
1. On Panel server: select "Panel Only". Save the public key (`SSL_CERT="..."`)
2. On Node server: select "Node only". Enter selfsteal domain, panel IP, and the saved public key

### Option 2: All-in-One (Panel and Node on one server)

Simplified option, suitable for testing or small loads.

**One server contains:**
- Remnawave Panel, Node, Subscription-Page
- Caddy, PostgreSQL, Redis

**DNS:** All three domains (different!) → IP of this single server

### Traffic Routing (All-in-One mode)

```
Client → Port 443 → Xray (local Remnawave Node)
                      ├─ (VLESS proxy traffic) → Processed by Xray
                      └─ (Non-VLESS traffic, fallback) → Caddy (port 9443)
                                                          ├─ SNI: Panel Domain → Remnawave Panel (port 3000)
                                                          ├─ SNI: Subscription Domain → Subscription Page (port 3010)
                                                          └─ SNI: Selfsteal Domain → Static HTML page
```

> **Note**: In All-in-One mode, if you stop the local Remnawave Node or break the Xray config, the panel and other web services will become inaccessible through domain names.

## Directory Structure

### Panel Installation (`/opt/remnawave/`)
```
/opt/remnawave/
├── .env                    # Panel environment
├── docker-compose.yml      # Panel services
├── credentials.txt         # Generated credentials
├── Makefile                # Service management
├── caddy/
│   ├── Caddyfile          # Caddy configuration
│   ├── docker-compose.yml
│   └── html/              # Static files
├── subscription-page/
│   └── docker-compose.yml
└── node/                   # (All-in-One only)
    ├── .env
    └── docker-compose.yml
```

### Separate Node Installation (`/opt/remnanode/`)
```
/opt/remnanode/
├── .env
├── docker-compose.yml
├── Makefile
└── selfsteal/
    ├── Caddyfile
    ├── docker-compose.yml
    └── html/
```

## Panel Access Protection

### SIMPLE Cookie Security
- Access via URL with secret key: `https://panel.example.com/auth/login?caddy=SECRET`
- Caddy sets cookie on first visit
- Without valid cookie/parameter, shows Selfsteal placeholder page

### FULL Caddy Security (recommended)
- Uses `remnawave/caddy-with-auth` image with `caddy-security` module
- **Two-level authentication:**
  1. Caddy Auth Portal (login/password + MFA on first login)
  2. Remnawave Panel login
- Panel accessible via random path: `https://panel.example.com/<RANDOM_PATH>/auth`

## Network Configuration

### Open Ports
| Port | Purpose |
|------|---------|
| 80/tcp | Caddy HTTP |
| 443/tcp | Caddy HTTPS / Xray |
| 22/tcp | SSH (or your custom port) |
| 2222/tcp | Node API (restricted) |

### Port 2222 Restrictions
- **All-in-One**: Open only for 172.30.0.0/16 (Docker subnet)
- **Separate Node**: Open only for panel IP
