# 📁 Containerised Nextcloud File Server

A fully containerised, self-hosted [Nextcloud](https://nextcloud.com/) file server powered by **Docker Compose**. Deploys three isolated containers — Nginx, Nextcloud (PHP-FPM), and MariaDB — with custom port configuration for Nginx (port 21) and MariaDB (port 3336).

> **Think Google Drive, but on your own server.**

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Host Machine                                   │  
│                                                                       │
│  ┌─────────────┐     ┌──────────────────┐    ┌───────────────┐   │
│  │    Nginx     │     │   Nextcloud App    │    │   MariaDB      │   │
│  │  (Alpine)    │───▶ │   (PHP-FPM)       │───▶│   (v11)        │   │
│  │              │     │                    │    │                │   │
│  │  Port: 21    │     │  Port: 9000        │    │  Port: 3336    │   │
│  │  (exposed)   │     │  (internal only)   │    │  (exposed)     │   │
│  └─────────────┘     └──────────────────┘    └───────────────┘   │
│         │                      │                         │           │
│         └────────────────────┴───────────────────────┘            │
│                    Docker Bridge Network                              │
│                   (nextcloud-net)                                     │
└─────────────────────────────────────────────────────────────────┘
```

### Container Breakdown

| Container | Image | Role | Port |
|-----------|-------|------|------|
| `nextcloud-web` | `nginx:alpine` | Reverse proxy — serves static files and forwards PHP requests to the app container | **21** (host-exposed) |
| `nextcloud-app` | `nextcloud:fpm` | Nextcloud application with PHP-FPM — handles all business logic | 9000 (internal) |
| `nextcloud-db` | `mariadb:11` | Database backend — stores users, files metadata, and settings | **3336** (host-exposed) |

### Request Flow

```
Browser Request
     │
     ▼
┌─────────┐     ┌──────────────┐     ┌──────────┐
│  Nginx  │────▶│  PHP-FPM     │────▶│ MariaDB    │
│  :21    │     │  (Nextcloud) │     │  :3336      │
│         │◀────│  :9000       │◀────│            │
└─────────┘     └──────────────┘     └──────────┘
     │
     ▼
Browser Response
```

1. **Browser** sends request to `http://nextcloud.fileshare.com:21`
2. **Nginx** handles the request:
   - Static files (CSS, JS, images) → served directly from the shared volume
   - PHP requests → forwarded to `nextcloud-app:9000` via FastCGI
3. **Nextcloud (PHP-FPM)** processes the request and queries **MariaDB** on port 3336
4. Response travels back through the chain to the browser

### Data Persistence

| Volume | Mount Point | Purpose |
|--------|-------------|---------|
| `db_data` | `/var/lib/mysql` | Database files |
| `nextcloud_data` | `/var/www/html` | Nextcloud application + user files |

Both volumes are Docker-managed named volumes, ensuring data survives container restarts and recreations.

---

## 📂 Project Structure

```
containerised-nextcloud/
├── docker-compose.yml   # Stack definition (3 services, 2 volumes, 1 network)
├── nginx.conf           # Nginx reverse proxy config (optimised for Nextcloud)
├── .env                 # Credentials ( not tracked in git)
├── .env.example         # Template for .env (safe to commit)
├── .gitignore           # Excludes .env and other sensitive/unnecessary files
└── README.md            # This file
```

---

## 🚀 Quick Start

### Prerequisites

- **Docker Engine** ≥ 24.0 with the **Compose plugin** (`docker compose`)
- A Linux host (tested on Ubuntu 26.04)
- Port **21** and **3336** available on the host

### Step 1 — Clone the Repository

```bash
git clone https://github.com/jacobtsajan/containerised_nextcloud_instance.git
cd containerised-nextcloud_instance
```

### Step 2 — Create the Environment File

```bash
cp .env.example .env
```

Edit `.env` and set **strong, unique passwords**:

```bash
nano .env
```

```env
# Database credentials
MYSQL_ROOT_PASSWORD=<your-secure-root-password>
MYSQL_DATABASE=nextcloud
MYSQL_USER=nextcloud
MYSQL_PASSWORD=<your-secure-db-password>

# Nextcloud admin credentials
NEXTCLOUD_ADMIN_USER=admin
NEXTCLOUD_ADMIN_PASSWORD=<your-secure-admin-password>
```

### Step 3 — Configure DNS

Add a hosts entry so the domain resolves to your server:

```bash
# On the server itself
echo "127.0.0.1 nextcloud.fileshare.com" | sudo tee -a /etc/hosts

# On any client machine that needs access (use the server's LAN IP)
echo "<server-ip> nextcloud.fileshare.com" | sudo tee -a /etc/hosts
```

### Step 4 — Allow Port 21 in Your Browser

Port 21 is blocked by default in most browsers (it's the traditional FTP port).

**Firefox:**
1. Navigate to `about:config`
2. Search for `network.security.ports.banned.override`
3. Create it as a **String** with value `21`

**Chrome/Chromium:**
```bash
# Launch with the flag
google-chrome --explicitly-allowed-ports=21
```

### Step 5 — Launch the Stack

```bash
docker compose up -d
```

Wait for all containers to become healthy (~30-60 seconds on first run):

```bash
docker compose ps
```

Expected output:
```
NAME             IMAGE           STATUS                    PORTS
nextcloud-db     mariadb:11      Up (healthy)              3306/tcp, 0.0.0.0:3336->3336/tcp
nextcloud-app    nextcloud:fpm   Up                        9000/tcp
nextcloud-web    nginx:alpine    Up                        0.0.0.0:21->21/tcp, 80/tcp
```

### Step 6 — Access Nextcloud

Open your browser and navigate to:

```
http://nextcloud.fileshare.com:21
```

Log in with the admin credentials you set in `.env`.

---

## ✅ Verification

Run these commands to confirm the setup meets requirements:

```bash
# Verify Nginx is listening on port 21
sudo ss -tlnp | grep ':21 '

# Verify MariaDB is listening on port 3336
sudo ss -tlnp | grep ':3336 '

# Test HTTP response
curl -I http://localhost:21

# Check Nextcloud status via occ
docker exec -u www-data nextcloud-app php occ status
```

---

## 🔧 Management

### Common Operations

```bash
# Stop all containers (data preserved)
docker compose down

# Stop and DELETE all data ( destructive!)
docker compose down -v

# Restart a specific service
docker compose restart web

# View logs
docker compose logs -f          # all services
docker compose logs -f web      # nginx only
docker compose logs -f app      # nextcloud only
docker compose logs -f db       # mariadb only

# Enter Nextcloud container shell
docker exec -it nextcloud-app bash

# Run Nextcloud occ commands
docker exec -u www-data nextcloud-app php occ <command>
```

### Useful occ Commands

```bash
# Check system status
docker exec -u www-data nextcloud-app php occ status

# Scan files (after manual upload)
docker exec -u www-data nextcloud-app php occ files:scan --all

# Add a trusted domain
docker exec -u www-data nextcloud-app php occ config:system:set trusted_domains 2 --value="new.domain.com"

# Enable maintenance mode
docker exec -u www-data nextcloud-app php occ maintenance:mode --on
```

---

## 🔒 Security Notes

- The `.env` file contains **sensitive credentials** and is excluded from version control via `.gitignore`
- Nginx is configured with security headers (X-Frame-Options, X-Content-Type-Options, CSP, etc.)
- The `X-Powered-By` header is stripped from responses
- Internal services (PHP-FPM on port 9000) are **not exposed** to the host — only accessible within the Docker network
- For production use, consider adding **HTTPS/TLS** via Let's Encrypt and a proper domain

---

## 🐛 Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Nginx container keeps restarting | Config syntax error | Check `docker logs nextcloud-web` for the exact error |
| "Requested URL not found" | Redirect missing port | Ensure `OVERWRITEHOST` includes `:21` in `docker-compose.yml` |
| "Address is restricted" (Firefox) | Browser blocks port 21 | Set `network.security.ports.banned.override` to `21` in `about:config` |
| "Untrusted domain" | Domain not in trusted list | Check `NEXTCLOUD_TRUSTED_DOMAINS` matches your access URL |
| 502 Bad Gateway | PHP-FPM not ready yet | Wait 30-60 seconds and refresh |
| DB connection refused | MariaDB still initializing | Wait for healthcheck to pass: `docker compose ps` should show `healthy` |

---

## 📝 License

This project is open source and available under the [MIT License](LICENSE).
