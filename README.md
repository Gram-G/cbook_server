# cbook_server

Repurposing an Acer Chromebook 15 (CB5-571, board: YUNA, Intel Celeron, x86_64) as a headless home server running Debian 13. The machine runs Pi-hole for network-wide DNS filtering and Samba for NAS functionality, accessible over the local network via SSH.

---

## Hardware

- **Device:** Acer Chromebook 15 CB5-571
- **Board:** YUNA
- **CPU:** Intel Celeron (x86_64)
- **RAM:** 3.7GB
- **Internal Storage:** 16GB (Kingston SSD)
- **External Storage:** 1TB USB HDD formatted ext4, mounted at `/mnt/external`
- **Router:** GL.iNet AX1800

---

## What Was Done

### 1. Unlocking the Chromebook
- Removed the write-protect screw from the motherboard
- Enabled Developer Mode
- Flashed MrChromebox UEFI Full ROM firmware to replace ChromeOS firmware entirely

### 2. OS Installation
- Installed Debian 13 (Trixie) with XFCE desktop via USB
- Configured user account with sudo access
- Installed and enabled OpenSSH server for remote management

### 3. Pi-hole v6
- Installed Pi-hole for network-wide ad and tracker blocking
- Configured the GL.iNet router DNS in **Manual DNS mode**:
  - Primary: `192.168.8.156` (Pi-hole)
  - Secondary: `1.1.1.1` (Cloudflare fallback)
- Router set to override DNS for all clients via DHCP

### 4. Samba NAS
- Installed Samba for Windows file sharing
- Formatted 1TB USB HDD to ext4, mounted at `/mnt/external`
- Configured fstab with UUID for reliable auto-mount on boot
- NAS share accessible at `\\192.168.8.156\NAS`
- Mapped as Z: drive on Windows PC

### 5. Network Hardening
- Disabled WiFi power saving to prevent connection dropouts:
  ```bash
  sudo iwconfig wlp1s0 power off
  ```
- UFW installed but disabled due to conflicts with Pi-hole DNS

---

## Problems Encountered and How They Were Fixed

### Problem: DNS fails after Chromebook reboot
**Symptom:** Windows PC showed "Connected" on Ethernet but couldn't browse. Other network devices also lost DNS resolution after a Chromebook reboot.

**Root cause:** The router was configured in DNS Proxy mode pointing exclusively at Pi-hole. When `pihole-FTL` hadn't fully started yet after a reboot, the router had no fallback DNS and the entire network lost name resolution.

**Fix 1 — Router fallback DNS:**
Switched router from DNS Proxy mode to Manual DNS mode, adding `1.1.1.1` as a secondary. Network stays functional even if Pi-hole is slow to start.

**Fix 2 — systemd service dependency:**
`pihole-FTL` was starting before the network interface was ready. Created a systemd drop-in override to enforce ordering:

```bash
sudo systemctl edit pihole-FTL
```

```ini
[Unit]
After=network-online.target
Wants=network-online.target
```

```bash
sudo systemctl enable NetworkManager-wait-online.service
sudo systemctl daemon-reload
sudo systemctl restart pihole-FTL
```

---

### Problem: WiFi not connecting until physical login
**Symptom:** After a reboot, SSH was unreachable until someone physically opened the Chromebook and logged in. The WiFi connection was stored as a user-level NetworkManager connection rather than a system-level one.

**Diagnosis:**
```bash
nmcli connection show
```
Identified the WiFi connection named `ATFSux` as user-scoped.

**Fix:**
```bash
sudo nmcli connection modify ATFSux connection.permissions ""
sudo nmcli connection up ATFSux
```

Removing the permissions field promotes it to a system connection that reconnects at boot without requiring a user session.

---

### Problem: Chromebook freezing and dropping SSH sessions
**Symptom:** SSH sessions dropped and the machine became unreachable. Required physical hard reboot to recover.

**Root cause 1 — System suspend:** The Chromebook was suspending due to inactivity. As a laptop, systemd had sleep and suspend targets active by default.

**Fix:**
```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

**Root cause 2 — Samba memory leak:** During one session, `smbd` consumed 2.9GB of RAM on a 3.7GB system, exhausting available memory and causing a freeze. This was a one-time leak during an unusually long session; normal Samba usage runs at ~10-30MB.

**Root cause 3 — Unnecessary GUI overhead:** LightDM and Xorg were running constantly at the login screen consuming ~200MB with no benefit on a headless server.

**Fix — Disable GUI entirely:**
```bash
sudo systemctl set-default multi-user.target
sudo reboot
```

Result: RAM usage at idle dropped from ~640MB to ~478MB. No Xorg, no LightDM, SSH access unchanged.

---

### Problem: NAS serving wrong drive capacity
**Symptom:** When copying files to Z:, Windows reported only ~16GB available instead of ~931GB. Properties showed the internal drive size, not the external HDD.

**Root cause:** On boot, Samba was starting before the external USB HDD was mounted. When `smbd` came up, `/mnt/external` was an empty directory on the 16GB internal drive. Samba shared that instead of the HDD.

**Fix — Samba dependency on external mount:**
```bash
sudo systemctl edit smbd
```

```ini
[Unit]
After=mnt-external.mount
Requires=mnt-external.mount
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart smbd
```

Samba now waits for the external drive to be mounted before starting. If the drive isn't present, Samba won't start rather than silently serving the wrong path.

---

### Problem: Z: drive dropping and failing to reconnect
**Symptom:** Z: drive disconnects and throws "local device name is already in use" on reconnect attempt.

**Fix — Clear stale mapping and remap:**
```powershell
net use Z: /delete
net use Z: \\192.168.8.156\NAS /user:user /persistent:yes
```

**Remaining mitigations (to implement):**
- Add `SO_KEEPALIVE` and `deadtime = 0` to `/etc/samba/smb.conf`
- Disable Windows SMB autodisconnect: `net config server /autodisconnect:-1`
- Add credentials to Windows Credential Manager for silent authentication

---

## Current Server State

| Service     | Status  | Notes                              |
|-------------|---------|-------------------------------------|
| SSH         | Running | Headless, accessible at boot        |
| pihole-FTL  | Running | DNS filtering, ~52MB RAM            |
| smbd        | Running | NAS share, ~10MB RAM                |
| GUI (XFCE)  | Disabled| multi-user.target, console only     |
| Suspend     | Masked  | All sleep targets disabled          |

---

## Network Layout

| Device          | IP              |
|-----------------|-----------------|
| Router          | 192.168.8.1     |
| Chromebook/Server | 192.168.8.156 |
| Main PC         | 192.168.8.141   |

---

## Lessons Learned

- systemd service ordering matters. `After=network-online.target` is not set by default on many services and needs to be explicitly added for anything that depends on the network being fully ready.
- On low-RAM machines, a GUI running at idle is not free. Disabling it on a headless server is always worth doing.
- DNS Proxy mode with a single upstream and no fallback is a single point of failure for the entire network. Always configure a secondary.
- Samba starts fast — faster than USB HDD mounts. Without an explicit dependency, it will serve whatever is at the mount point at startup, which may not be what you expect.
