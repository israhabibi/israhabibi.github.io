---
title: "2-Laptop HA Homelab: Auto-Failover with Cloudflare + Syncthing"
date: 2025-10-20
authors:
  - isra
categories:
  - Homelab
tags:
  - homelab
  - high-availability
  - cloudflare
  - syncthing
  - docker
  - pi-hole
  - self-hosted
---

# 2-Laptop HA Homelab: Auto-Failover with Cloudflare + Syncthing

Two old laptops, one primary and one failover, running Immich + Flask apps over Cloudflare Tunnel — with automatic DNS failover in 30–60 seconds when the primary drops, plus Jellyfin, Pi-hole, and Portainer for local-only services.

<!-- more -->

## Context

I had two old laptops sitting around (8 GB and 4 GB RAM). Rather than treat either as disposable, I wired them up as a primary/secondary pair: the 8 GB one runs the live internet services, the 4 GB one mirrors the data and steps in when the primary goes dark. Failover is automatic for anything exposed via Cloudflare; local-only services stay on whichever node hosts them.

---

## Hardware

| Role | Specs |
|---|---|
| Laptop 1 (Primary) | 8 GB RAM, `192.168.1.101` |
| Laptop 2 (Failover) | 4 GB RAM, `192.168.1.102` |
| Personal laptop | Client device |

---

## Service Distribution

| Service | Laptop 1 | Laptop 2 | Access | Failover |
|---|---|---|---|---|
| Immich | Primary | Replica | Internet (Cloudflare) | Auto (30–60 s) |
| Flask apps | Primary | Replica | Internet (Cloudflare) | Auto (30–60 s) |
| Jellyfin | Active | Active | Local only | Manual |
| Pi-hole | — | Primary | Local only | — |
| Portainer | Active | — | Local only | Manual |
| Traefik | Active | Active | Local only | — |
| Homepage | Active | — | Local only | Manual |
| Syncthing | Active | Active | Background | — |

Key principle: **only internet-facing services need automatic failover**. Local-only services (Jellyfin, Pi-hole) either run on both or are manual — home users can tolerate a manual switch.

---

## Architecture

```
                           Internet
                              │
                       Cloudflare Edge
                              │
               ┌──────────────┴──────────────┐
               │                             │
          Tunnel 1 (Active)            Tunnel 2 (Standby)
               │                             │
──────────────┼─────────────────────────────┼──────────────
               │       Home Network (LAN)    │
     ┌─────────┴──────────┐         ┌────────┴──────────┐
     │  Laptop 1 (8 GB)   │         │  Laptop 2 (4 GB)  │
     │  192.168.1.101     │◄───────►│  192.168.1.102    │
     │                    │ Syncthing│                  │
     │ Immich / Flask     │         │ Immich (replica) │
     │ Jellyfin / Traefik │         │ Jellyfin / Pi-hole│
     │ Portainer / Home   │         │ Failover script   │
     └────────────────────┘         └───────────────────┘
```

---

## Laptop 1 — Primary (Active)

### Cloudflared Config

**`~/.cloudflared/config.yml`**

```yaml
tunnel: <TUNNEL_1_UUID>
credentials-file: /home/user/.cloudflared/<TUNNEL_1_UUID>.json

ingress:
  - hostname: immich.yourdomain.com
    service: http://localhost:2283
  - hostname: app.yourdomain.com
    service: http://localhost:5000
  - hostname: api.yourdomain.com
    service: http://localhost:5001
  - service: http_status:404
```

```bash
sudo cloudflared service install
sudo systemctl enable --now cloudflared
```

### Folder Layout

```
~/docker/
├── immich/           # internet-facing
├── flask-app/        # internet-facing
├── jellyfin/         # local only
├── portainer/        # local only
├── homepage/         # local only
└── traefik/          # local reverse proxy
~/data/               # synced via Syncthing
├── immich/photos
├── jellyfin/media
└── app-data/
```

---

## Laptop 2 — Failover (Standby)

Same folder shape, same `~/.cloudflared/config.yml` (but pointed at Tunnel 2), and the cloudflared service is **disabled by default**:

```bash
sudo cloudflared service install
sudo systemctl disable --now cloudflared
```

The failover script below starts it when needed.

---

## Data Replication with Syncthing

```bash
sudo apt install syncthing
sudo systemctl enable --now syncthing@$USER
```

Web UI: `http://laptop1.local:8384` and `http://laptop2.local:8384`.

Configure three send-and-receive folders:

| Folder | Path | Versioning |
|---|---|---|
| Immich photos | `~/data/immich/photos` | Staggered 30 days |
| Jellyfin media | `~/data/jellyfin/media` | Staggered 30 days |
| App data | `~/data/app-data` | Ignore `*.log`, `*.tmp`, `cache/` |

On Laptop 1, add Laptop 2 as a remote device; accept on Laptop 2; share each folder.

---

## Auto-Failover Script

Runs on Laptop 2. Polls Laptop 1's health every 30 seconds; on failure, starts cloudflared and updates Cloudflare DNS records to point at Tunnel 2.

**`~/scripts/auto-failover.sh`**

```bash
#!/bin/bash
LAPTOP1_IP="192.168.1.101"
CF_ZONE_ID="your_cloudflare_zone_id"
CF_RECORD_ID_IMMICH="immich_record_id"
CF_RECORD_ID_APP="app_record_id"
CF_API_TOKEN="your_api_token"
TUNNEL1_CNAME="tunnel-1-uuid.cfargotunnel.com"
TUNNEL2_CNAME="tunnel-2-uuid.cfargotunnel.com"

FAILOVER_TRIGGERED=false
LOG_FILE="/var/log/homelab-failover.log"

log() { echo "[$(date '+%F %T')] $1" | tee -a "$LOG_FILE"; }

check_immich() {
  curl -sf --max-time 5 "http://${LAPTOP1_IP}:2283/api/server-info" > /dev/null
}
check_flask() {
  curl -sf --max-time 5 "http://${LAPTOP1_IP}:5000/health" > /dev/null
}

update_cf_dns() {
  curl -X PUT "https://api.cloudflare.com/client/v4/zones/$CF_ZONE_ID/dns_records/$1" \
    -H "Authorization: Bearer $CF_API_TOKEN" \
    -H "Content-Type: application/json" \
    --data "{\"type\":\"CNAME\",\"content\":\"$2\",\"proxied\":true}" \
    -s > /dev/null
}

log "Starting auto-failover monitor"
while true; do
  if ! check_immich || ! check_flask; then
    if [ "$FAILOVER_TRIGGERED" = false ]; then
      log "ALERT: Laptop 1 down. Failing over to Laptop 2..."
      sudo systemctl start cloudflared
      sleep 5
      update_cf_dns "$CF_RECORD_ID_IMMICH" "$TUNNEL2_CNAME"
      update_cf_dns "$CF_RECORD_ID_APP"    "$TUNNEL2_CNAME"
      FAILOVER_TRIGGERED=true
      log "SUCCESS: Failover complete"
    fi
  else
    if [ "$FAILOVER_TRIGGERED" = true ]; then
      log "INFO: Laptop 1 recovered. Failing back..."
      update_cf_dns "$CF_RECORD_ID_IMMICH" "$TUNNEL1_CNAME"
      update_cf_dns "$CF_RECORD_ID_APP"    "$TUNNEL1_CNAME"
      sleep 10
      sudo systemctl stop cloudflared
      FAILOVER_TRIGGERED=false
      log "SUCCESS: Failback complete"
    fi
  fi
  sleep 30
done
```

### Install as systemd Service

**`/etc/systemd/system/homelab-failover.service`**

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

```bash
chmod +x ~/scripts/auto-failover.sh
sudo systemctl daemon-reload
sudo systemctl enable --now homelab-failover
sudo journalctl -u homelab-failover -f
```

---

## Local DNS with Pi-hole

Pi-hole runs on Laptop 2 as the primary local DNS.

```bash
curl -sSL https://install.pi-hole.net | bash
```

**`/etc/pihole/custom.list`**

```
192.168.1.101 laptop1.local
192.168.1.101 jellyfin.local
192.168.1.101 portainer.local
192.168.1.101 home.local
192.168.1.101 immich.local

192.168.1.102 laptop2.local
192.168.1.102 jellyfin2.local
192.168.1.102 pihole.local
```

```bash
pihole restartdns
```

Point the router's DHCP DNS at `192.168.1.102` so every device gets Pi-hole automatically.

---

## Docker Compose Examples

### Traefik (Local Reverse Proxy)

```yaml
services:
  traefik:
    image: traefik:v2.10
    container_name: traefik
    restart: unless-stopped
    ports: ["80:80"]
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    command:
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80
    networks: [homelab]

networks:
  homelab:
    name: homelab
    driver: bridge
```

### Jellyfin

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Jakarta
    volumes:
      - ./config:/config
      - /home/user/data/jellyfin/media:/media
    ports: ["8096:8096"]
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.jellyfin.rule=Host(`jellyfin.local`)"
      - "traefik.http.services.jellyfin.loadbalancer.server.port=8096"
    networks: [homelab]

networks:
  homelab:
    external: true
```

---

## Access Cheat Sheet

| Service | Internet URL | Local URL |
|---|---|---|
| Immich | `https://immich.yourdomain.com` | `http://immich.local` |
| Flask app | `https://app.yourdomain.com` | — |
| Jellyfin | — (by design) | `http://jellyfin.local` |
| Portainer | — | `http://portainer.local` |
| Pi-hole | — | `http://pihole.local/admin` |
| Homepage | — | `http://home.local` |

Jellyfin is deliberately **not** internet-exposed — streaming video through Cloudflare's free tier is against ToS and burns residential upstream.

---

## Maintenance Rhythm

| Cadence | Task |
|---|---|
| Daily | Check Syncthing status; `journalctl -u homelab-failover --since today` |
| Weekly | `docker ps -a`, verify Cloudflare tunnel, Syncthing completion |
| Monthly | `docker-compose pull && up -d`, test failover, clean `.stversions`, `pihole -up` |

---

## Troubleshooting

!!! bug "Failover didn't trigger"
    `sudo journalctl -u homelab-failover -n 50` and manually test the health endpoints: `curl http://192.168.1.101:2283/api/server-info`. If the script thinks Laptop 1 is up, your health check URL is wrong.

!!! bug "Syncthing not syncing"
    Check (1) port 22000 open between laptops, (2) folder IDs match, (3) disk space. Web UI at `:8384` usually tells you which one.

!!! bug "Local DNS not resolving"
    `nslookup jellyfin.local 192.168.1.102` from a client. If that works but the browser doesn't, the client isn't using Pi-hole — fix DHCP on the router.

---

## Security

- Only Immich + Flask are internet-exposed (Cloudflare proxy = DDoS + HTTPS for free).
- Jellyfin, Portainer, Pi-hole admin stay LAN-only.
- UFW on both laptops: `allow from 192.168.1.0/24`, deny everything else.
- Consider Tailscale/WireGuard for remote admin instead of exposing SSH.

---

## Key Takeaways

- **Auto-failover only where it matters.** Local services don't need it; internet-facing ones do. Scope keeps the moving parts small.
- **Cloudflare DNS updates are the failover mechanism**, not DNS TTLs — proxied records update in seconds when you PATCH via the API.
- **Syncthing bidirectional sync is plenty** for photos and media at this scale. You don't need DRBD or a SAN.
- **Pi-hole doubles as a DNS and a local name service** — `*.local` entries in one file make the whole LAN navigable without a reverse proxy per box.
- **The 4 GB failover node works** because standby containers are stopped; it only powers up cloudflared + replicas when the primary dies.
