# Homelab High Availability Setup Documentation

## Overview

2-laptop homelab setup with mixed access (internet + local only), automatic failover for internet services, and data replication.

**Hardware:**
- Laptop 1: 8GB RAM (Primary)
- Laptop 2: 4GB RAM (Secondary/Failover)
- Personal Laptop: Client device

---

## Architecture

### Network Topology

```
Internet Users
     ↓
Cloudflare Tunnel (Immich + Flask Apps Only)
     ↓
Laptop 1 (Primary) ←--Auto Failover--→ Laptop 2 (Standby)
     ↑                                        ↑
     └────────── Local Network ──────────────┘
              (All Services Available)
```

### Service Distribution

| Service | Laptop 1 (8GB) | Laptop 2 (4GB) | Access Method | Failover |
|---------|----------------|----------------|---------------|----------|
| **Immich** | Primary | Replica | Internet (Cloudflare) | Auto (30-60s) |
| **Flask Apps** | Primary | Replica | Internet (Cloudflare) | Auto (30-60s) |
| **Jellyfin** | Active | Active | Local Only | Manual |
| **Pi-hole** | - | Primary | Local Only | Manual |
| **Portainer** | Active | - | Local Only | Manual |
| **Traefik** | Active | Active | Local Only | - |
| **Homepage** | Active | - | Local Only | Manual |
| **Syncthing** | Active | Active | Background | - |

---

## Laptop 1 Configuration (8GB RAM - Primary)

### Services Breakdown

**Internet-Facing (via Cloudflare Tunnel):**
- Immich (primary instance)
- Flask web applications
- Cloudflared tunnel → `immich.yourdomain.com`, `app.yourdomain.com`

**Local-Only:**
- Jellyfin → `http://laptop1.local:8096` or `http://jellyfin.local`
- Traefik (local reverse proxy) → `http://laptop1.local`
- Portainer → `http://laptop1.local:9000`
- Homepage dashboard → `http://laptop1.local:3000`

**Background Services:**
- Syncthing (data replication to Laptop 2)
- Docker runtime

### Folder Structure

```
/home/user/
├── docker/
│   ├── immich/              # Exposed via Cloudflare
│   │   └── docker-compose.yml
│   ├── flask-app/           # Exposed via Cloudflare
│   │   └── docker-compose.yml
│   ├── jellyfin/            # Local only
│   │   └── docker-compose.yml
│   ├── portainer/           # Local only
│   │   └── docker-compose.yml
│   ├── homepage/            # Local only
│   │   └── docker-compose.yml
│   └── traefik/             # Local reverse proxy
│       └── docker-compose.yml
├── data/                    # Synced via Syncthing
│   ├── immich/
│   │   ├── photos/
│   │   └── database/
│   ├── jellyfin/
│   │   └── media/
│   └── app-data/
└── .cloudflared/
    ├── config.yml
    └── <tunnel-uuid>.json
```

### Cloudflared Configuration

**File:** `~/.cloudflared/config.yml`

```yaml
tunnel: <TUNNEL_1_UUID>
credentials-file: /home/user/.cloudflared/<TUNNEL_1_UUID>.json

ingress:
  # Immich - exposed to internet
  - hostname: immich.yourdomain.com
    service: http://localhost:2283
    
  # Flask app - exposed to internet
  - hostname: app.yourdomain.com
    service: http://localhost:5000
    
  # API endpoint
  - hostname: api.yourdomain.com
    service: http://localhost:5001
  
  # Catch-all (required)
  - service: http_status:404
```

**Start cloudflared:**
```bash
# Install as service
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```

---

## Laptop 2 Configuration (4GB RAM - Secondary)

### Services Breakdown

**Internet-Facing (standby - inactive by default):**
- Immich (replica instance)
- Flask apps (replica instances)
- Cloudflared tunnel (stopped, starts on failover)

**Local-Only:**
- Jellyfin → `http://laptop2.local:8096` or `http://jellyfin2.local`
- Traefik (local reverse proxy)
- Pi-hole (primary DNS) → `http://laptop2.local:8080` or `http://pihole.local`

**Background Services:**
- Syncthing (receives data from Laptop 1)
- Auto-failover monitoring script

### Folder Structure

```
/home/user/
├── docker/
│   ├── immich/              # Replica (same config)
│   ├── flask-app/           # Replica (same config)
│   ├── jellyfin/            # Local only
│   ├── pihole/              # Local DNS
│   └── traefik/
├── data/                    # Synced from Laptop 1
│   ├── immich/
│   ├── jellyfin/
│   └── app-data/
├── .cloudflared/
│   ├── config.yml           # Same as Laptop 1
│   └── <tunnel-uuid>.json
└── scripts/
    └── auto-failover.sh     # Monitoring script
```

### Cloudflared Configuration

**File:** `~/.cloudflared/config.yml`

```yaml
tunnel: <TUNNEL_2_UUID>
credentials-file: /home/user/.cloudflared/<TUNNEL_2_UUID>.json

# Same ingress config as Laptop 1
ingress:
  - hostname: immich.yourdomain.com
    service: http://localhost:2283
    
  - hostname: app.yourdomain.com
    service: http://localhost:5000
    
  - hostname: api.yourdomain.com
    service: http://localhost:5001
  
  - service: http_status:404
```

**Setup (but don't start by default):**
```bash
# Install but disable service
sudo cloudflared service install
sudo systemctl disable cloudflared
sudo systemctl stop cloudflared

# Will be started by auto-failover script when needed
```

---

## Data Replication with Syncthing

### Setup on Both Laptops

**Install Syncthing:**
```bash
# Ubuntu/Debian
sudo apt install syncthing

# Enable service
sudo systemctl enable syncthing@$USER
sudo systemctl start syncthing@$USER
```

**Access Web UI:**
- Laptop 1: `http://laptop1.local:8384`
- Laptop 2: `http://laptop2.local:8384`

### Shared Folders Configuration

Create these shared folders (bidirectional sync):

1. **Immich Photos**
   - Path: `/home/user/data/immich/photos`
   - Sync Type: Send & Receive
   - Version Control: Staggered (30 days)

2. **Jellyfin Media**
   - Path: `/home/user/data/jellyfin/media`
   - Sync Type: Send & Receive
   - Version Control: Staggered (30 days)

3. **Application Data**
   - Path: `/home/user/data/app-data`
   - Sync Type: Send & Receive
   - Ignore Patterns: `*.log`, `*.tmp`, `cache/`

### Connect Laptops

1. On Laptop 1: Add Remote Device (use Laptop 2's Device ID)
2. On Laptop 2: Accept connection request
3. Share folders between devices

---

## Auto-Failover System

### Monitoring Script

**File:** `/home/user/scripts/auto-failover.sh`

```bash
#!/bin/bash
# Auto-failover for internet-facing services only
# Run this on Laptop 2

# Configuration
LAPTOP1_IP="192.168.1.101"
CF_ZONE_ID="your_cloudflare_zone_id"
CF_RECORD_ID_IMMICH="immich_record_id"
CF_RECORD_ID_APP="app_record_id"
CF_API_TOKEN="your_api_token"
TUNNEL1_CNAME="tunnel-1-uuid.cfargotunnel.com"
TUNNEL2_CNAME="tunnel-2-uuid.cfargotunnel.com"

FAILOVER_TRIGGERED=false
LOG_FILE="/var/log/homelab-failover.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Health check functions
check_immich() {
    curl -sf --max-time 5 "http://${LAPTOP1_IP}:2283/api/server-info" > /dev/null 2>&1
    return $?
}

check_flask() {
    curl -sf --max-time 5 "http://${LAPTOP1_IP}:5000/health" > /dev/null 2>&1
    return $?
}

update_cf_dns() {
    local record_id=$1
    local target_cname=$2
    
    curl -X PUT "https://api.cloudflare.com/client/v4/zones/$CF_ZONE_ID/dns_records/$record_id" \
        -H "Authorization: Bearer $CF_API_TOKEN" \
        -H "Content-Type: application/json" \
        --data "{\"type\":\"CNAME\",\"content\":\"$target_cname\",\"proxied\":true}" \
        -s > /dev/null
}

# Main monitoring loop
log "Starting auto-failover monitor"

while true; do
    if ! check_immich || ! check_flask; then
        if [ "$FAILOVER_TRIGGERED" = false ]; then
            log "ALERT: Laptop 1 services down. Initiating failover to Laptop 2..."
            
            # Start cloudflared on Laptop 2
            sudo systemctl start cloudflared
            sleep 5
            
            # Update DNS records to Laptop 2's tunnel
            update_cf_dns "$CF_RECORD_ID_IMMICH" "$TUNNEL2_CNAME"
            update_cf_dns "$CF_RECORD_ID_APP" "$TUNNEL2_CNAME"
            
            FAILOVER_TRIGGERED=true
            log "SUCCESS: Failover to Laptop 2 completed"
            
            # Send notification (optional)
            # curl -X POST "https://ntfy.sh/yourtopic" -d "Homelab failed over to Laptop 2"
        fi
    else
        # Laptop 1 is healthy
        if [ "$FAILOVER_TRIGGERED" = true ]; then
            log "INFO: Laptop 1 recovered. Initiating failback..."
            
            # Update DNS back to Laptop 1
            update_cf_dns "$CF_RECORD_ID_IMMICH" "$TUNNEL1_CNAME"
            update_cf_dns "$CF_RECORD_ID_APP" "$TUNNEL1_CNAME"
            
            sleep 10
            
            # Stop cloudflared on Laptop 2
            sudo systemctl stop cloudflared
            
            FAILOVER_TRIGGERED=false
            log "SUCCESS: Failback to Laptop 1 completed"
            
            # Send notification (optional)
            # curl -X POST "https://ntfy.sh/yourtopic" -d "Homelab failed back to Laptop 1"
        fi
    fi
    
    sleep 30  # Check every 30 seconds
done
```

### Install as Systemd Service

**File:** `/etc/systemd/system/homelab-failover.service`

```ini
[Unit]
Description=Homelab Auto-Failover Monitor
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=youruser
ExecStart=/bin/bash /home/youruser/scripts/auto-failover.sh
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**Enable and start:**
```bash
sudo chmod +x /home/user/scripts/auto-failover.sh
sudo systemctl daemon-reload
sudo systemctl enable homelab-failover
sudo systemctl start homelab-failover

# Check status
sudo systemctl status homelab-failover

# View logs
sudo journalctl -u homelab-failover -f
```

---

## Local DNS with Pi-hole

### Pi-hole Setup (Laptop 2)

**Install Pi-hole:**
```bash
curl -sSL https://install.pi-hole.net | bash
```

**Configuration:**
- Interface: Listen on all interfaces
- Upstream DNS: Cloudflare (1.1.1.1)
- Admin Web Port: 8080

### Local DNS Records

**File:** `/etc/pihole/custom.list`

```
# Laptop 1 services
192.168.1.101 laptop1.local
192.168.1.101 jellyfin.local
192.168.1.101 portainer.local
192.168.1.101 home.local
192.168.1.101 immich.local

# Laptop 2 services
192.168.1.102 laptop2.local
192.168.1.102 jellyfin2.local
192.168.1.102 pihole.local
```

**Apply changes:**
```bash
pihole restartdns
```

### Configure Clients

**Option 1: Router-level (Recommended)**
- Set DHCP DNS server to Laptop 2 IP: `192.168.1.102`
- All devices automatically use Pi-hole

**Option 2: Per-device**
- Manually set DNS to `192.168.1.102`

---

## Docker Compose Examples

### Traefik (Local Reverse Proxy)

**File:** `~/docker/traefik/docker-compose.yml`

```yaml
version: '3.8'

services:
  traefik:
    image: traefik:v2.10
    container_name: traefik
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    command:
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80
    networks:
      - homelab

networks:
  homelab:
    name: homelab
    driver: bridge
```

### Jellyfin (Local Only)

**File:** `~/docker/jellyfin/docker-compose.yml`

```yaml
version: '3.8'

services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Jakarta
    volumes:
      - ./config:/config
      - /home/user/data/jellyfin/media:/media
    ports:
      - "8096:8096"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.jellyfin.rule=Host(`jellyfin.local`)"
      - "traefik.http.services.jellyfin.loadbalancer.server.port=8096"
    networks:
      - homelab

networks:
  homelab:
    external: true
```

### Immich (Internet + Local)

**File:** `~/docker/immich/docker-compose.yml`

```yaml
version: '3.8'

services:
  immich-server:
    image: ghcr.io/immich-app/immich-server:release
    container_name: immich_server
    restart: unless-stopped
    command: ['start.sh', 'immich']
    volumes:
      - /home/user/data/immich/photos:/usr/src/app/upload
      - /etc/localtime:/etc/localtime:ro
    environment:
      - DB_HOSTNAME=immich_postgres
      - DB_USERNAME=postgres
      - DB_PASSWORD=postgres
      - DB_DATABASE_NAME=immich
      - REDIS_HOSTNAME=immich_redis
    ports:
      - "2283:3001"
    depends_on:
      - immich_redis
      - immich_postgres
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.immich.rule=Host(`immich.local`)"
      - "traefik.http.services.immich.loadbalancer.server.port=3001"
    networks:
      - homelab

  immich_redis:
    image: redis:6.2-alpine
    container_name: immich_redis
    restart: unless-stopped
    networks:
      - homelab

  immich_postgres:
    image: tensorchord/pgvecto-rs:pg14-v0.2.0
    container_name: immich_postgres
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_USER: postgres
      POSTGRES_DB: immich
    volumes:
      - ./postgres:/var/lib/postgresql/data
    networks:
      - homelab

networks:
  homelab:
    external: true
```

---

## Access Methods

### Internet Access (From Anywhere)

| Service | URL | Notes |
|---------|-----|-------|
| Immich | `https://immich.yourdomain.com` | Auto-failover enabled |
| Flask App | `https://app.yourdomain.com` | Auto-failover enabled |
| API | `https://api.yourdomain.com` | Auto-failover enabled |

### Local Access (Home Network Only)

| Service | Primary URL | Backup URL |
|---------|-------------|------------|
| Jellyfin | `http://jellyfin.local` | `http://jellyfin2.local` |
| Immich | `http://immich.local` | `http://laptop2.local:2283` |
| Portainer | `http://portainer.local` | `http://laptop1.local:9000` |
| Pi-hole | `http://pihole.local/admin` | `http://laptop2.local:8080/admin` |
| Homepage | `http://home.local` | `http://laptop1.local:3000` |

---

## Personal Laptop Workflow

### Immich Photo Sync

**Option 1: Immich CLI (Recommended)**
```bash
# Install
npm install -g @immich/cli

# Configure
immich login https://immich.yourdomain.com

# Upload photos
immich upload ~/Pictures/
```

**Option 2: zsync (Your current method)**
```bash
# Sync to Laptop 1
zsync ~/Photos/ user@laptop1.local:/home/user/data/immich/photos/

# Syncthing will automatically replicate to Laptop 2
```

### Jellyfin Access

**When at home:**
```bash
# Open browser
firefox http://jellyfin.local
```

**Not accessible from internet** (by design - security & bandwidth)

---

## Maintenance Tasks

### Daily
- Monitor Syncthing sync status
- Check failover script logs: `journalctl -u homelab-failover --since today`

### Weekly
- Check Docker container health: `docker ps -a`
- Review Cloudflare tunnel status
- Verify backup completion in Syncthing

### Monthly
- Update Docker images: `docker-compose pull && docker-compose up -d`
- Test failover by stopping Laptop 1 services
- Clean old Syncthing versions: Check `.stversions` folders
- Update Pi-hole: `pihole -up`

---

## Troubleshooting

### Failover Not Working

**Check monitoring script:**
```bash
sudo systemctl status homelab-failover
sudo journalctl -u homelab-failover -n 50
```

**Test health endpoints manually:**
```bash
curl http://192.168.1.101:2283/api/server-info
curl http://192.168.1.101:5000/health
```

**Check Cloudflare tunnel on Laptop 2:**
```bash
sudo systemctl status cloudflared
sudo cloudflared tunnel list
```

### Syncthing Not Syncing

**Check Syncthing status:**
```bash
# Web UI
http://laptop1.local:8384
http://laptop2.local:8384

# Check service
sudo systemctl status syncthing@$USER

# Check logs
journalctl -u syncthing@$USER -f
```

**Common issues:**
- Firewall blocking port 22000
- Different folder IDs between devices
- Insufficient disk space

### Local DNS Not Resolving

**Check Pi-hole:**
```bash
# Status
pihole status

# Restart DNS
pihole restartdns

# Check custom DNS
cat /etc/pihole/custom.list
```

**Verify client DNS settings:**
```bash
# Linux/Mac
cat /etc/resolv.conf

# Should show: nameserver 192.168.1.102
```

### Jellyfin Performance Issues

**Check transcoding:**
- Disable transcoding for local network
- Settings → Playback → "Direct Play" preferred

**Check media sync:**
```bash
# Verify Syncthing completed
du -sh /home/user/data/jellyfin/media
```

---

## Security Considerations

### Internet-Facing Services
- ✅ Only Immich and Flask apps exposed
- ✅ Cloudflare proxy provides DDoS protection
- ✅ HTTPS enforced via Cloudflare
- ⚠️ Enable Immich authentication (password required)
- ⚠️ Enable Flask app authentication if needed

### Local Services
- ✅ Jellyfin not exposed to internet (bandwidth + security)
- ✅ Pi-hole admin interface password-protected
- ✅ Portainer requires authentication
- ⚠️ Consider VPN (Tailscale/WireGuard) for remote admin access

### Network Security
```bash
# Firewall rules (UFW example)
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.1.0/24  # Local network
sudo ufw enable
```

---

## Backup Strategy

### What to Backup

1. **Critical Data (Daily via Syncthing):**
   - Immich photos (already replicated)
   - Jellyfin media (already replicated)
   - Application databases

2. **Configuration Files (Weekly):**
   - Docker compose files
   - Cloudflared configs
   - Scripts (auto-failover.sh)
   - Pi-hole custom DNS

3. **External Backup (Monthly):**
   ```bash
   # Backup to external drive
   rsync -av /home/user/data/ /mnt/external/homelab-backup/
   ```

### Automated Backup Script

```bash
#!/bin/bash
# backup.sh - Run weekly via cron

BACKUP_DIR="/mnt/external/homelab-backup"
DATE=$(date +%Y%m%d)

# Backup configs
tar -czf "$BACKUP_DIR/configs-$DATE.tar.gz" \
    ~/docker/ \
    ~/.cloudflared/ \
    ~/scripts/ \
    /etc/pihole/custom.list

# Keep only last 4 backups
cd "$BACKUP_DIR"
ls -t configs-*.tar.gz | tail -n +5 | xargs rm -f
```

**Setup cron:**
```bash
crontab -e

# Add line:
0 2 * * 0 /home/user/scripts/backup.sh
```

---

## Quick Reference Commands

### Docker Management
```bash
# View all containers
docker ps -a

# Restart all services
cd ~/docker/immich && docker-compose restart

# View logs
docker logs -f immich_server

# Update images
docker-compose pull && docker-compose up -d

# Clean up
docker system prune -a
```

### Service Management
```bash
# Cloudflare tunnel
sudo systemctl status cloudflared
sudo systemctl restart cloudflared

# Failover monitor
sudo systemctl status homelab-failover
sudo journalctl -u homelab-failover -f

# Syncthing
sudo systemctl status syncthing@$USER
```

### Network Diagnostics
```bash
# Test local DNS
nslookup jellyfin.local 192.168.1.102

# Test Cloudflare tunnel
curl -I https://immich.yourdomain.com

# Check ports
sudo netstat -tulpn | grep LISTEN

# Test failover manually
sudo systemctl stop docker  # On Laptop 1
# Wait 30-60 seconds, check if Laptop 2 takes over
```

---

## Cloudflare Configuration

### Create Tunnels

**Laptop 1:**
```bash
cloudflared tunnel create homelab-1
cloudflared tunnel route dns homelab-1 immich.yourdomain.com
cloudflared tunnel route dns homelab-1 app.yourdomain.com
```

**Laptop 2:**
```bash
cloudflared tunnel create homelab-2
cloudflared tunnel route dns homelab-2 immich.yourdomain.com
cloudflared tunnel route dns homelab-2 app.yourdomain.com
```

### Get API Token

1. Go to Cloudflare Dashboard
2. My Profile → API Tokens → Create Token
3. Edit zone DNS template
4. Copy token for auto-failover script

### Get Zone and Record IDs

```bash
# Get Zone ID
curl -X GET "https://api.cloudflare.com/client/v4/zones" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  | jq -r '.result[] | select(.name=="yourdomain.com") | .id'

# Get DNS Record IDs
curl -X GET "https://api.cloudflare.com/client/v4/zones/ZONE_ID/dns_records" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  | jq -r '.result[] | select(.name=="immich.yourdomain.com") | .id'
```

---

## Appendix: Network Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        Internet                              │
│                            │                                 │
│                   Cloudflare Edge                            │
│                            │                                 │
│              ┌─────────────┴──────────────┐                  │
│              │                            │                  │
│         Tunnel 1 (Active)          Tunnel 2 (Standby)        │
└──────────────┼────────────────────────────┼─────────────────┘
               │                            │
               │     Home Network (LAN)     │
               │                            │
    ┌──────────┴─────────┐       ┌──────────┴─────────┐
    │   Laptop 1 (8GB)   │       │   Laptop 2 (4GB)   │
    │    192.168.1.101   │◄─────►│    192.168.1.102   │
    ├────────────────────┤       ├────────────────────┤
    │ Internet Services: │       │ Internet Services: │
    │ • Immich           │       │ • Immich (replica) │
    │ • Flask Apps       │       │ • Flask (replica)  │
    │                    │       │                    │
    │ Local Services:    │       │ Local Services:    │
    │ • Jellyfin         │       │ • Jellyfin         │
    │ • Portainer        │       │ • Pi-hole          │
    │ • Homepage         │       │ • Failover Script  │
    └────────┬───────────┘       └─────────┬──────────┘
             │                             │
             │      Syncthing Sync         │
             │◄───────────────────────────►│
             │                             │
             │                             │
    ┌────────┴─────────────────────────────┴──────────┐
    │            Other Devices (Phones, PC)           │
    │  • Access via Pi-hole DNS                       │
    │  • Local: http://jellyfin.local                 │
    │  • Remote: https://immich.yourdomain.com        │
    └─────────────────────────────────────────────────┘
```

---

## Document Information

- **Created:** 2025-10-20
- **Last Updated:** 2025-10-20
- **Version:** 1.0
- **Author:** Homelab Admin
- **Status:** Active Configuration

---

## Change Log

| Date | Version | Changes |
|------|---------|---------|
| 2025-10-20 | 1.0 | Initial documentation |

---

## Additional Resources

- **Cloudflare Tunnel Docs:** https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/
- **Docker Compose Reference:** https://docs.docker.com/compose/
- **Syncthing Documentation:** https://docs.syncthing.net/
- **Pi-hole Documentation:** https://docs.pi-hole.net/
- **Immich Documentation:** https://immich.app/docs/overview/introduction
- **Jellyfin Documentation:** https://jellyfin.org/docs/

---

## Notes

- Always test failover in a controlled manner before relying on it
- Keep this documentation updated when making infrastructure changes
- Regularly review logs for any anomalies
- Consider adding monitoring notifications (ntfy.sh, Telegram, etc.)
- Document any custom modifications to this setup