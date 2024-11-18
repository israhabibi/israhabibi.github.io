---
layout: post
title:  "Avahi + Traefik for Subdomains"
date:   2024-11-18 +0700
categories: Web
---

### **Steps to Set Up Avahi + Traefik**

---

#### **1. Set Up Avahi for `gws.local`**
1. **Install Avahi**:
   ```bash
   sudo apt update
   sudo apt install avahi-daemon
   ```

2. **Configure Avahi Hostname**:
   - Edit the hostname for your server:
     ```bash
     sudo nano /etc/hostname
     ```
     Set it to `gws`.
   - Update `/etc/hosts` with your desired domain:
     ```bash
     sudo nano /etc/hosts
     ```
     Example:
     ```
     127.0.0.1       localhost
     192.168.1.100   gws.local gws
     ```

3. **Restart Avahi**:
   ```bash
   sudo systemctl restart avahi-daemon
   ```

   Test by pinging `gws.local` from another device:
   ```bash
   ping gws.local
   ```

---

#### **2. Install Traefik**
1. **Deploy Traefik with Docker Compose**:
   Create a `docker-compose.yml` file for Traefik:
   ```yaml
   version: '3.7'
   services:
     traefik:
       image: traefik:v2.10
       container_name: traefik
       command:
         - "--api.insecure=true"
         - "--providers.docker=true"
         - "--entrypoints.web.address=:80"
       ports:
         - "80:80"
         - "8080:8080" # Optional: Traefik dashboard
       volumes:
         - "/var/run/docker.sock:/var/run/docker.sock:ro"
         - "./traefik.yml:/traefik.yml:ro"
       restart: unless-stopped
   ```

2. **Start Traefik**:
   ```bash
   docker-compose up -d
   ```

3. **Test Traefik**:
   - Access the dashboard (optional): `http://gws.local:8080`.

---

#### **3. Map Subdomains with Traefik**
To set up `phome.gws.local`, configure your service and Traefik labels.

1. **Edit Your Service's Docker Compose**:
   For example, if you're running Pi-hole:
   ```yaml
   version: '3.7'
   services:
     pihole:
       image: pihole/pihole:latest
       container_name: pihole
       environment:
         TZ: "Your/Timezone"
       labels:
         - "traefik.http.routers.pihole.rule=Host(`phome.gws.local`)"
         - "traefik.http.services.pihole.loadbalancer.server.port=80"
       networks:
         - traefik_proxy
       restart: unless-stopped
   networks:
     traefik_proxy:
       external: true
   ```

2. **Update Traefik's Network**:
   Ensure all services, including Traefik, are on the same Docker network:
   ```bash
   docker network create traefik_proxy
   ```

3. **Test Access**:
   Access `http://phome.gws.local` from your browser.

---

#### **4. Update Avahi for Subdomains**
Avahi doesn’t natively support nested subdomains, but you can add aliases.

1. **Create a Custom Avahi Service File**:
   ```bash
   sudo nano /etc/avahi/services/traefik.service
   ```
   Example:
   ```xml
   <service-group>
     <name replace-wildcards="yes">gws.local</name>
     <service>
       <type>_http._tcp</type>
       <port>80</port>
       <txt-record>path=/</txt-record>
     </service>
     <service>
       <name>phome.gws.local</name>
       <type>_http._tcp</type>
       <port>80</port>
     </service>
   </service-group>
   ```

2. **Restart Avahi**:
   ```bash
   sudo systemctl restart avahi-daemon
   ```

---

### **Testing**
- Open `http://gws.local` for the main server.
- Open `http://phome.gws.local` to access Pi-hole or another service.

---

### **Why This Setup Works**
- **Avahi**: Handles local hostname resolution (`gws.local` and `phome.gws.local`).
- **Traefik**: Acts as a reverse proxy to route subdomains to specific services.

Let me know if you run into any issues!