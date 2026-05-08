# SSH Recovery & Tailscale Race Condition Fix

A practical guide for regaining SSH access to a remote Linux server (Ubuntu / CasaOS / Docker) when you are locked out due to a Tailscale bind race or a firewall/CrowdSec ban.

---

## The Problem

Two independent failures can leave you locked out:

| Failure | Cause | Effect |
|---|---|---|
| **SSH bind failure** | `sshd` starts before `tailscale0` exists | SSH listens only on the local interface, not the Tailscale IP |
| **Firewall/CrowdSec ban** | Your remote IP triggers a ban rule | All TCP port 22 traffic is silently dropped |

---

## Emergency Recovery — The "Trojan Horse" Container

If you still have access to a web-based Docker manager (CasaOS, Portainer, etc.) you can deploy a privileged container that shares the host's namespaces and use it to fix the host from the inside.

### Step 1 — Deploy the rescue container

Paste this into your Docker manager as a new stack/service:

```yaml
services:
  rescue:
    image: ubuntu:latest
    container_name: rescue_ssh
    network_mode: host
    pid: host
    privileged: true
    tty: true
    stdin_open: true
```

### Step 2 — Break into the host namespace

Open a terminal inside the `rescue_ssh` container and run:

```bash
nsenter -t 1 -m -u -n -i bash
```

`nsenter -t 1` targets PID 1 (the host's init process). After this command you are in a **root shell on the host**, not inside the container.

### Step 3 — Unban yourself (iptables)

Insert an ACCEPT rule at the top of the INPUT chain so port 22 is always reachable, without breaking Docker's NAT rules below:

```bash
apt update && apt install iptables -y
iptables -I INPUT 1 -p tcp --dport 22 -j ACCEPT
```

You should now be able to SSH in normally.

---

## Permanent Fix — Systemd Service Override

The root cause is that `ssh.service` starts before `tailscaled.service` finishes and assigns the Tailscale IP. Fix it with a drop-in override:

```bash
sudo systemctl edit ssh.service
```

Paste the following and save:

```ini
[Unit]
After=network-online.target tailscaled.service
Wants=network-online.target tailscaled.service

[Service]
Restart=on-failure
RestartSec=5s
```

Reload and apply:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh
```

SSH will now wait for both the network and Tailscale to be fully online before starting, and will automatically restart if it still fails.

---

## Security Cleanup

The rescue container is a critical security hole — it gives anyone with Docker access full root on the host. Destroy it immediately after you are done:

```bash
docker stop rescue_ssh && docker rm rescue_ssh
```

---

## Summary

```
Locked out
    │
    ├─ CasaOS/Portainer still accessible?
    │       │
    │       └─ YES → Deploy rescue container → nsenter → fix iptables → SSH in
    │
    └─ Apply systemd override so this never happens again
```

---

## Verified Output — Before & After

### Before (broken — race condition)

`sshd` starts at boot before `tailscale0` is assigned an IP. It silently skips the Tailscale interface and only listens on the LAN. If you are connecting remotely over Tailscale, you are locked out.

```
● ssh.service - OpenBSD Secure Shell server

     Loaded: loaded (/usr/lib/systemd/system/ssh.service; enabled; preset: enabled)
     Active: active (running) since Fri 2026-05-08 13:33:59 UTC; 5h 44min ago
    Process: ExecStartPre=/usr/sbin/sshd -t (code=exited, status=0/SUCCESS)
   Main PID: (sshd)
     Memory: 260.0K (peak: 2.4M swap: 1.0M swap peak: 1.0M)
     CGroup: /system.slice/ssh.service
             └─ "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"

May 08 13:33:59 taskmanager systemd[1]: Starting ssh.service - OpenBSD Secure Shell server...
May 08 13:33:59 taskmanager sshd: error: Bind to port 22 on <tailscale-ip> failed: Cannot assign requested address
May 08 13:33:59 taskmanager sshd: Server listening on <local-ip> port 22.
May 08 13:33:59 taskmanager systemd[1]: Started ssh.service - OpenBSD Secure Shell server.
```

> Note: systemd reports `Active: active (running)` even though the Tailscale bind failed — the service is not considered failed because it still has one working listener. This makes the bug easy to miss.

---

### After (fixed — systemd override applied)

Both interfaces are bound successfully on startup:

```
root@taskmanager:/# systemctl restart ssh

● ssh.service - OpenBSD Secure Shell server

     Loaded: loaded (/usr/lib/systemd/system/ssh.service; enabled; preset: enabled)
     Active: active (running) since Fri 2026-05-08 19:19:12 UTC; 2s ago
    Process: ExecStartPre=/usr/sbin/sshd -t (code=exited, status=0/SUCCESS)
   Main PID: (sshd)
     Memory: 1.7M (peak: 2.0M)
     CGroup: /system.slice/ssh.service
             └─ "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"

May 08 19:19:12 taskmanager systemd[1]: Starting ssh.service - OpenBSD Secure Shell server...
May 08 19:19:12 taskmanager sshd: Server listening on <tailscale-ip> port 22.
May 08 19:19:12 taskmanager sshd: Server listening on <local-ip> port 22.
May 08 19:19:12 taskmanager systemd[1]: Started ssh.service - OpenBSD Secure Shell server.
```

The `error: Bind to port 22 ... failed` line is gone and both `Server listening on` lines are present.

---

## Tested On

- Ubuntu 22.04 / 24.04
- CasaOS with Docker
- Tailscale (any version)
