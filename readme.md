# sovereignty (Gitea + Nextcloud + OnlyOffice + WireGuard)

This repository deploys an SME-friendly, cloud-independent collaboration stack.

## What you get
- Gitea (Git hosting)
- Nextcloud (file sync/share)
- OnlyOffice Document Server (collaborative editing)
- PostgreSQL + Redis
- WireGuard VPN-first access
- Optional Caddy reverse proxy with internal HTTPS (TLS internal)

## Security model
- Only UDP WireGuard is exposed publicly.
- App ports bind to the WireGuard interface IP, not 0.0.0.0.

## Quick start (Debian 13)
1) Copy env template and edit secrets:
```bash
cp .env.example .env
nano .env
