---
title: "Setting Up an Old Ubuntu Desktop as a Home Server"
date: 2026-04-10
authors:
  - isra
tags:
  - homelab
  - ubuntu
  - linux
  - docker
  - self-hosted
---

# Setting Up an Old Ubuntu Desktop as a Home Server

Got an old Ubuntu desktop collecting dust? Here's how to repurpose it into a lean home server — disabling the GUI, locking in a static IP, setting up SSH, and managing containers with Portainer.

<!-- more -->

## Context

I had an old machine running Ubuntu Desktop that I wanted to convert into a local server for self-hosted tools. The goal: strip it down to headless mode, make it accessible over the network, and get Docker containers running cleanly.

---

## Step 1 — Disable the Desktop GUI

No need to uninstall the desktop environment — just disable the display manager so it doesn't load on boot.

```bash
# Disable the display manager (GNOME example)
sudo systemctl disable gdm3

# Set boot target to non-graphical
sudo systemctl set-default multi-user.target
```

After reboot, Ubuntu boots straight to TTY — no GUI loaded, no wasted RAM.

**RAM impact:**

| State | Estimated RAM usage |
|---|---|
| Ubuntu Desktop (GUI active) | ~800 MB – 1.5 GB |
| After disabling GUI | ~200 – 400 MB |

> If you ever want the GUI back: `sudo systemctl set-default graphical.target`

If you're sure you'll never need the desktop again, you can also fully uninstall it:

```bash
sudo apt remove --purge ubuntu-desktop gnome-shell
sudo apt autoremove
```

---

## Step 2 — Enable SSH

Make the server accessible from your other machines:

```bash
sudo apt install openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```

### Troubleshooting: Connection Timeout

If `ssh user@192.168.x.x` times out, work through these checks in order:

**1. Can you ping the server?**
```bash
ping 192.168.x.x
```
If this times out too, the problem is network-level, not SSH.

**2. Check the firewall (UFW)**
```bash
sudo ufw status
sudo ufw allow ssh   # if active
```

**3. Verify SSH is actually running**
```bash
sudo systemctl status ssh
```

**4. Confirm the server's IP (it may have changed)**
```bash
ip a | grep inet
```
If you're on DHCP, the IP likely changed since you last checked. Fix this in Step 3.

---

## Step 3 — Set a Static IP

A server with a changing IP is annoying. Set a static address via Netplan:

```bash
# Check your interface name first
ip a

# Edit the Netplan config
sudo nano /etc/netplan/01-netcfg.yaml
```

Example config for a static IP:

```yaml
network:
  version: 2
  ethernets:
    eth0:           # replace with your interface name
      dhcp4: no
      addresses:
        - 192.168.1.100/24
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

Apply the config:

```bash
sudo netplan apply
```

---

## Step 4 — Audit Running Services

Before installing anything, identify what's already running and trim the fat:

```bash
# All currently running services
systemctl list-units --type=service --state=running

# All enabled services (auto-start on boot)
systemctl list-unit-files --type=service --state=enabled
```

Common candidates to disable on a headless server:

```bash
sudo systemctl disable bluetooth      # no bluetooth peripherals
sudo systemctl disable cups           # no printer
sudo systemctl disable avahi-daemon   # no mDNS service discovery needed
sudo systemctl disable ModemManager   # no USB modem
sudo systemctl disable snapd          # if not using snap packages
```

Always check what a service does before disabling:
```bash
systemctl status <service-name>
```

---

## Step 5 — Install Docker

```bash
# Install Docker Engine
sudo apt update
sudo apt install -y docker.io

# Start and enable Docker
sudo systemctl enable docker
sudo systemctl start docker

# Allow your user to run Docker without sudo
sudo usermod -aG docker $USER
```

Log out and back in for the group change to take effect.

---

## Step 6 — Choose a Container Manager: Portainer vs CasaOS

Both are useful, but serve different purposes:

| | CasaOS | Portainer |
|---|---|---|
| **Target** | Home server / NAS-like | Developer / sysadmin |
| **UI** | All-in-one dashboard, app store | Docker-focused control panel |
| **Best for** | Click-to-install self-hosted apps | Full control over containers, networks, volumes |
| **Complexity** | Low | Medium (requires Docker familiarity) |

For a setup where you already have Docker installed and want detailed control, **Portainer** is the better fit. Install it as a container:

```bash
docker run -d \
  -p 9000:9000 \
  --name portainer \
  --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce
```

Access it at `http://<your-server-ip>:9000`.

If you installed CasaOS and want to remove it cleanly:

```bash
curl -fsSL https://get.casaos.io/uninstall | sudo bash
```

---

## Step 7 — Organize Docker Containers with Compose

Rather than running containers one-by-one with `docker run`, use Docker Compose with one folder per service:

```
~/docker/
├── portainer/
│   └── docker-compose.yml
├── nextcloud/
│   └── docker-compose.yml
└── jellyfin/
    └── docker-compose.yml
```

Each service has its own `docker-compose.yml`. To manage any service:

```bash
cd ~/docker/portainer
docker compose up -d      # start
docker compose down       # stop
docker compose pull       # update image
```

This structure is easy to back up, reproduce, and extend as you add more services.

---

## Step 8 — Monitor CPU Temperature (Optional)

If the machine is older, keep an eye on thermals. Install `lm-sensors` and run:

```bash
sudo apt install lm-sensors
sudo sensors-detect
sensors
```

For a live terminal dashboard:

```bash
sudo apt install s-tui
s-tui
```

Healthy idle temps: below 60°C for most CPUs. If you're seeing 90°C+ at idle, check for dust in the heatsink or dried thermal paste.

---

## Key Takeaways

- Disabling the GUI frees ~1 GB of RAM without touching your packages — it's the right first move
- SSH connection timeouts are almost always a firewall or IP change issue, not an SSH issue
- Set a static IP before doing anything else — saves a lot of future headaches
- Portainer is the right pick if you're already Docker-familiar and want container-level control
- Docker Compose + one folder per service is the cleanest way to manage a growing homelab
