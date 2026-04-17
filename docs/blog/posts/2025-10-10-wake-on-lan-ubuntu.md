---
title: "Wake-on-LAN on an Ubuntu Laptop Server"
date: 2025-10-10
authors:
  - isra
categories:
  - Homelab
tags:
  - homelab
  - ubuntu
  - linux
  - wake-on-lan
  - networking
---

# Wake-on-LAN on an Ubuntu Laptop Server

Turn on your Ubuntu laptop-server remotely with a magic packet — useful when it's tucked away and you don't want it drawing power 24/7.

<!-- more -->

## Context

I use an old laptop as a home server, but I don't want it running full-time. Wake-on-LAN (WOL) lets me send a "wake up" network packet from my phone or another machine and have the server boot itself up — no physical button press needed.

Laptops are trickier than desktops: you need AC power, an Ethernet connection (Wi-Fi WOL is rarely reliable), and a BIOS that supports the feature. Here's the full setup.

---

## Prerequisites

- Laptop plugged into AC (battery-mode WOL is usually disabled by firmware)
- Ethernet cable (not Wi-Fi)
- Ubuntu Server (or Desktop) installed
- Another device on the same network to send the magic packet

---

## Step 1 — Enable WOL in BIOS/UEFI

Reboot and enter BIOS. The option is typically under **Power Management** and might be named:

- `Wake on LAN`
- `Power on by PCI-E`
- `Wake on PCI Event`

Enable it, save, exit.

!!! warning "If you don't see the option"
    Some consumer laptops bury WOL or omit it entirely. Check the laptop's service manual before going further.

---

## Step 2 — Install `ethtool`

```bash
sudo apt update
sudo apt install ethtool
```

Find your Ethernet interface:

```bash
ip a
```

Look for something like `eno1`, `enp3s0`, or `eth0` with an `inet` address.

Verify the NIC supports WOL:

```bash
sudo ethtool <interface>
```

Look for:

```
Supports Wake-on: pumbg
Wake-on: d
```

`Supports Wake-on: g` means "magic packet" is supported. `Wake-on: d` means it's currently **disabled**.

---

## Step 3 — Enable WOL on the Interface

```bash
sudo ethtool -s <interface> wol g
```

Confirm:

```bash
sudo ethtool <interface> | grep Wake-on
# Wake-on: g
```

This change is **not persistent** — it'll reset on reboot. Fix that in Step 4.

---

## Step 4 — Make WOL Persistent via systemd

Create a unit file:

```bash
sudo nano /etc/systemd/system/wol@.service
```

```ini
[Unit]
Description=Enable Wake-on-LAN on %i
Requires=network.target
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/ethtool -s %i wol g
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable it for your interface (replace `enp3s0` with yours):

```bash
sudo systemctl daemon-reload
sudo systemctl enable wol@enp3s0
sudo systemctl start wol@enp3s0
```

The `@` template means you can reuse this service for any interface name by passing it after the `@`.

---

## Step 5 — Get the MAC Address

From the server:

```bash
ip link show <interface> | grep ether
# link/ether aa:bb:cc:dd:ee:ff brd ff:ff:ff:ff:ff:ff
```

Write down the MAC address — you'll need it to send the magic packet.

---

## Step 6 — Handle the Closed-Lid Problem

By default, closing the laptop lid triggers suspend, and suspended laptops don't always respond to WOL reliably. Force the lid close to be ignored:

```bash
sudo nano /etc/systemd/logind.conf
```

Change (or add):

```
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

Reload:

```bash
sudo systemctl restart systemd-logind
```

Now the laptop stays awake with the lid closed — which is what you want for a headless server.

---

## Step 7 — Send a Magic Packet

From **another device** on the same network:

```bash
# Ubuntu / Debian
sudo apt install wakeonlan
wakeonlan aa:bb:cc:dd:ee:ff

# macOS
brew install wakeonlan
wakeonlan aa:bb:cc:dd:ee:ff
```

Shut the server down fully (`sudo poweroff`), wait a moment, then send the packet. If everything is wired correctly, the server powers on within a few seconds.

---

## Verification Checks

**Is sleep mode available?**

```bash
cat /sys/power/mem_sleep
# [s2idle] deep   ← deep sleep supported
```

**Are the Ethernet LEDs still lit after shutdown?**

Look at the physical NIC after `sudo poweroff`. If the LEDs stay on, the NIC is receiving standby power — which is required for WOL to work at all.

If the LEDs go dark, the BIOS isn't keeping the NIC powered. Re-check Step 1.

---

## Constraints to Know

- **AC power required** — almost all laptop firmware disables WOL when running on battery.
- **Ethernet only** — Wi-Fi WOL (WoWLAN) exists but is flaky; stick to wired.
- **Same broadcast domain** — magic packets are broadcast-only by default, so the sender must be on the same subnet. For WOL over the internet, set up a port-forward + broadcast-relay on your router, or SSH into a local always-on device (like a Pi-hole) and trigger WOL from there.

---

## Key Takeaways

- `ethtool -s <if> wol g` is the core command, but it's not persistent — wrap it in a systemd unit.
- Closed-lid suspend defeats WOL; override it in `/etc/systemd/logind.conf`.
- Ethernet LEDs staying lit after shutdown = NIC has standby power = WOL will work.
- For remote WOL from outside your LAN, use a small always-on device as a relay instead of exposing broadcast traffic.
