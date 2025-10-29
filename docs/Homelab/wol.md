# Wake-on-LAN (WOL) Setup on Ubuntu Laptop Server

This guide explains how to enable and configure **Wake-on-LAN (WOL)** on a laptop running Ubuntu Server.

---

## 🧩 1. Check BIOS/UEFI Support

1. Reboot your laptop and enter the BIOS/UEFI setup (usually **F2**, **Del**, or **Esc**).
2. Look for and enable one of the following:
   - **Wake on LAN**
   - **Power on by PCI-E**
   - **Wake from Network**
3. Save and exit.

> ⚠️ Note: Many laptops only support WOL when **plugged into AC power** (not on battery).

---

## ⚙️ 2. Enable WOL in Ubuntu

Install `ethtool`:
```bash
sudo apt update
sudo apt install ethtool
```

Check your network interface:
```bash
ip link
```

Verify WOL capability:
```bash
sudo ethtool <interface>
```

Enable WOL:
```bash
sudo ethtool -s <interface> wol g
```

Confirm:
```bash
sudo ethtool <interface> | grep Wake-on
```

Expected output:
```
Wake-on: g
```

---

## 🔁 3. Make WOL Persistent

Create a systemd service:
```bash
sudo nano /etc/systemd/system/wol@.service
```

Add:
```ini
[Unit]
Description=Wake-on-LAN for %i
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/ethtool -s %i wol g

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
sudo systemctl enable wol@<interface>.service
sudo systemctl start wol@<interface>.service
```

---

## 🌐 4. Get MAC Address

```bash
ip link show <interface> | grep ether
```

Example:
```
ether 00:11:22:33:44:55
```

---

## ⚡ 5. Test WOL

On another computer in the same LAN:

Install wakeonlan:
```bash
sudo apt install wakeonlan
```

Send the magic packet:
```bash
wakeonlan 00:11:22:33:44:55
```

The laptop should power on if:
- It’s **connected via Ethernet**
- **Plugged into power**
- **Properly configured in BIOS**

---

## 💻 6. Laptop Lid Behavior

By default, closing the lid puts the laptop to sleep, disabling WOL.  
To prevent this:

Edit logind configuration:
```bash
sudo nano /etc/systemd/logind.conf
```

Set:
```
HandleLidSwitch=ignore
HandleLidSwitchDocked=ignore
```

Apply changes:
```bash
sudo systemctl restart systemd-logind
```

### Lid Guidelines

| Scenario | Works with Lid Closed? | Notes |
|-----------|------------------------|-------|
| Shutdown + AC Power | ✅ | WOL works fine |
| Suspend (Sleep) | ⚠️ | Depends on hardware |
| On Battery Only | ❌ | Usually unsupported |
| Running Server Mode | ✅ | Use `HandleLidSwitch=ignore` |

---

## 🧠 Notes & Tips

- **WOL over Wi-Fi** is generally **not supported** on laptops.
- To check suspend support:
  ```bash
  cat /sys/power/mem_sleep
  ```
  - `s2idle`: WOL won't work
  - `deep`: Might work if NIC remains powered
- Ensure Ethernet LEDs remain lit after shutdown (indicates NIC still powered).

---

## ✅ TL;DR

1. Enable WOL in BIOS and Ubuntu.
2. Keep laptop **on AC power**.
3. Use **Ethernet** (not Wi-Fi).
4. Set:
   ```
   HandleLidSwitch=ignore
   ```
   to run with lid closed.
5. Test with:
   ```bash
   wakeonlan <MAC_ADDRESS>
   ```

---
