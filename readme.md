
---

## Network Design (WireGuard)

### Principles
- Use WireGuard for **admin and internal traffic**, not as a replacement for public HTTPS.
- Keep **Nextcloud/OnlyOffice publicly reachable** via HTTPS; keep **admin endpoints** private via WG.
- Prefer a **hub-and-spoke** topology with the hosted server as the hub (simpler than full mesh).

### Addressing (example)
- wg0 subnet: `10.9.0.0/24`
  - hosted server: `10.9.0.1`
  - backup A: `10.9.0.11`
  - backup B: `10.9.0.12`
  - your laptop: `10.9.0.21`
  - employees: `10.9.0.50-10.9.0.99`

### Routing
- Do **not** route general internet traffic through WG by default.
- Allow only:
  - admin SSH to server
  - backup pull endpoints
  - optional internal access to Nextcloud/OnlyOffice by WG DNS name

### Access Control
- Firewall on hosted server:
  - Public: `443/tcp` only
  - WG: `22/tcp` (admin), backup ports as needed
- Use per-peer rules (AllowedIPs) + server firewall to prevent lateral movement.

---

## Hosted Server: Service Layout

### Components
- Reverse Proxy: **nginx** (or Caddy/Traefik)  
  - Terminates TLS (Let’s Encrypt)
  - Routes:
    - `cloud.example.tld` → Nextcloud
    - `office.example.tld` → OnlyOffice Document Server
- Nextcloud stack:
  - Nextcloud (PHP-FPM)
  - DB (Postgres preferred)
  - Redis for file locking + caching
- OnlyOffice Document Server:
  - Typically deployed as Docker container
  - Connected to Nextcloud via the OnlyOffice app + shared secret (JWT)

### Data Storage
- Keep Nextcloud data directory on a dedicated volume:
  - e.g. `/srv/nextcloud/data`
- Keep DB on dedicated volume:
  - e.g. `/srv/db`
- Keep configuration and secrets separated:
  - `/etc/nextcloud/`
  - `/etc/onlyoffice/`

---

## Backup Design (On-prem Pull, Read-only)

### Threat Model
- Hosted server compromise should not automatically compromise backups.
- Therefore: **pull-based backups** with:
  - A restricted account on the hosted server
  - Read-only access to required paths
  - No interactive shell or minimal forced commands

### Backup Content
- Nextcloud data directory
- Nextcloud config (`config.php`)
- Database dumps (or physical backups)
- OnlyOffice config (if customized)
- Reverse proxy config + cert state (optional; can be regenerated)

### Implementation Pattern
- On hosted server:
  - Create a dedicated backup user `backup_pull`
  - Restrict via:
    - `authorized_keys` forced command
    - `rbash` or command wrappers
    - filesystem ACLs (read-only)
- On backup machines:
  - Schedule `restic`/`borg` pull over SSH via WireGuard
  - Store backups encrypted
  - Apply retention policies

### Suggested Mechanics (example)
- DB: nightly `pg_dump` to `/srv/backup/db_dump.sql.gz` (root-owned, readable by backup user only)
- Files: restic/borg pulls:
  - `/srv/nextcloud/data`
  - `/etc/nextcloud/`
  - `/srv/backup/db_dump.sql.gz`

### Integrity & Restore Tests
- Monthly automated restore test to a disposable VM/container:
  - Import DB dump
  - Mount/sync data
  - Run Nextcloud `occ maintenance:repair`

---

## IaC / Reproducible Provisioning

### Objectives
- Rebuild the hosted server from scratch in < 1 hour:
  - Provision base OS
  - Apply hardening + firewall
  - Deploy services
  - Restore from backup

### Recommended Split
1. **Provisioning** (server baseline)
   - (Optional) Terraform if the hosting provider supports it
   - Otherwise: a documented manual “create Debian VM” step
2. **Configuration Management** (the main IaC)
   - **Ansible** as the primary tool (idempotent, easy to maintain)
3. **Application deployment**
   - Prefer **Docker Compose** for OnlyOffice + optional Nextcloud
   - Or native packages for nginx + PHP + Postgres if you want fewer moving parts

### Ansible Role Structure (suggested)
- `roles/base` (users, ssh hardening, unattended upgrades)
- `roles/wireguard` (wg0, peers, firewall)
- `roles/nginx` (TLS, vhosts)
- `roles/nextcloud` (php, config, occ tasks)
- `roles/onlyoffice` (docker compose, JWT secret)
- `roles/backup_exports` (backup user, db dump job, permissions)
- `roles/monitoring` (optional)

### Secrets Handling
- Use Ansible Vault (or sops) for:
  - JWT secret
  - DB passwords
  - admin credentials bootstrap
- Keep secrets out of Git.

---

## Offline Usage on Your Laptop (Mirrored Data + OnlyOffice)

### Data Mirroring
- Use Nextcloud desktop client:
  - Sync selected folders to a local directory (e.g. `~/Nextcloud`)
  - Configure selective sync for “offline-critical” datasets

### OnlyOffice Offline Options

#### Option A (Most Practical): OnlyOffice Desktop Editors
- Pros:
  - Truly offline, no local server complexity
  - Edits local files directly
- Cons:
  - Not the same as “OnlyOffice server” workflow

#### Option B (Requested): Local OnlyOffice Document Server on Laptop
- Run local docserver via Docker on laptop
- Use it for:
  - Local-only editing sessions via browser
  - (Advanced) integrate with a *local* Nextcloud instance (heavy) or direct file endpoints
- Caveats:
  - Document Server is designed to be called by an integrator (Nextcloud/OwnCloud/etc.)
  - Running docserver standalone for editing local files is not as smooth as Desktop Editors
  - If you require full parity with online integration, you’d likely need:
    - a local Nextcloud (or compatible) connector
    - or accept Desktop Editors as the offline editor

### Recommended Practical Compromise
- Mirror data via Nextcloud client
- Use OnlyOffice Desktop Editors offline
- Keep OnlyOffice Document Server only on hosted server for collaborative web editing

---

## Operational Policies

### Accounts & Access
- Enforce per-user accounts in Nextcloud (no sharing passwords)
- Require MFA for Nextcloud admin users
- Separate admin group from normal users
- Use SSH keys + WireGuard for admin access; no password SSH

### Updates
- Enable unattended security updates on Debian
- Schedule maintenance windows for:
  - Nextcloud major upgrades
  - OnlyOffice upgrades
- Snapshot / backup before upgrades

### Monitoring (Minimal but Effective)
- Service health checks (nginx, php-fpm, db, redis, onlyoffice)
- Disk space alerting (data volume + backup volume)
- Certificate expiry alert

---

## Failure Modes & Recovery (Sketch)
- Hosted server loss:
  - Re-provision Debian VM
  - Run Ansible playbook
  - Restore from on-prem backups
- Accidental deletion / ransomware in Nextcloud:
  - Restore from restic/borg snapshots
  - Consider immutable storage on backups (append-only / WORM-like policy)
- WireGuard peer compromise:
  - Revoke peer key (remove from server config)
  - Rotate credentials/secrets if needed

---

## Open Design Decisions (to finalize)
- Hosting provider / VM snapshot capabilities
- Nextcloud deployment method:
  - native packages vs docker
- DB choice:
  - Postgres vs MariaDB
- Backup tooling:
  - restic vs borg
- Offline editing approach:
  - Desktop Editors vs local docserver + connector

---
End of sketch.
