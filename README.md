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

## Tested On

- Ubuntu 22.04 / 24.04
- CasaOS with Docker
- Tailscale (any version)
