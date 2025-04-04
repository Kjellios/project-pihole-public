# README_DEV.md

# Developer Notes – Pi-hole in Docker (Raspberry Pi)

This document provides internal guidance and context for the design, configuration, and operational decisions behind this Pi-hole Docker deployment on a Raspberry Pi 3. It is not intended for end-users setting up the system — they should refer to `README.md`.

---

## Project Purpose

The goal of this repository is to run Pi-hole inside a Docker container on a Raspberry Pi 3 using Raspberry Pi OS Lite (Legacy, 32-bit). All configuration data is persisted locally, network-level DNS functionality is enabled via host networking, and the system is designed to survive reboots and container rebuilds.

This serves both as a production-ready network ad blocker and as a learning project for containerized deployments on resource-constrained devices.

---

## Architecture Overview

### Host System

- **Device**: Raspberry Pi 3 Model B
- **OS**: Raspberry Pi OS Lite (Legacy, 32-bit)
- **Networking**: Wired Ethernet only (Wi-Fi disabled for stability)

### Container Runtime

- **Docker CE** installed via official script
- **Docker Compose** installed via APT package manager
- **No port mappings** — container uses `network_mode: host`

### Directory Layout (inside repo)

```
.
├── docker-compose.yml
├── etc-pihole/           # Mounted Pi-hole persistent config
├── etc-dnsmasq/          # Mounted DNSMasq override config
├── README.md             # End-user setup guide
├── README_DEV.md         # Developer/internal notes (this file)
├── TROUBLESHOOTING.md    # Bug-fix guide for common Pi-hole issues
```

---

## Container Configuration Notes

### Docker Networking

- `network_mode: "host"` is **required** to allow the container to bind to port 53 on the host.
- This removes the need for manual port mappings, but restricts this solution to Linux-based Docker hosts only.

### Environment Variables (defined in `docker-compose.yml`)

```yaml
environment:
  TZ: America/Chicago
  WEBPASSWORD: ${PIHOLE_WEBPASSWORD:-changeme}
  DNSMASQ_LISTENING: local
  PIHOLE_DNS_1: 1.1.1.1
  PIHOLE_DNS_2: 9.9.9.9
  DNSMASQ_USER: root
```

#### Rationale:
- `TZ` ensures accurate log timestamps and time-based features.
- `WEBPASSWORD` is defined using a default/fallback pattern (`:-changeme`).
- `DNSMASQ_LISTENING=local` restricts DNS resolution to the Pi.
- `DNSMASQ_USER=root` is **required** when using host networking; without this, DNS will silently fail.
- Upstream DNS servers are explicitly configured (Cloudflare and Quad9) to ensure reliable resolution and avoid ISP DNS.

---

## Volume Mounts & Permissions

### Volumes

```yaml
volumes:
  - ./etc-pihole:/etc/pihole
  - ./etc-dnsmasq:/etc/dnsmasq.d
```

This ensures persistent Pi-hole settings, adlists, query logs, and DNSMasq override rules are stored on the host filesystem.

### Permissions

These commands must be run before starting the container:

```sh
sudo chown -R 1000:1000 ./etc-pihole ./etc-dnsmasq
sudo chmod -R 755 ./etc-pihole ./etc-dnsmasq
```

Why:
- UID `1000` matches the default user created in Raspberry Pi OS.
- These permissions allow the container (running as root) to read/write to mounted directories reliably.

---

## DNS Settings Behavior

- The Pi-hole container stores DNS settings in `setupVars.conf`, located in `/etc/pihole`.
- If `DNSMASQ_LISTENING` is not set **both** in `setupVars.conf` and in `docker-compose.yml`, it will reset on container restart.
- All DNS configuration values should be mirrored in the environment block to ensure persistence across lifecycle events.

---

## Development & Debugging Commands

### Start / Stop

```sh
docker-compose up -d
docker-compose down
docker-compose down -v   # also deletes volumes (use with caution)
```

### Logs & Access

```sh
docker logs pihole
docker exec -it pihole bash
docker exec -it pihole cat /etc/pihole/setupVars.conf
```

### Config Testing

```sh
docker exec -it pihole pihole status
docker exec -it pihole pihole -d
```

---

## Version Management

### Current Image

- `pihole/pihole:latest`

### Plan

Eventually, pin to a specific version tag (e.g. `pihole/pihole:v2024.03.2`) to avoid accidental breakage due to upstream changes.

> Note: Some versions of Pi-hole may require migration steps for newer config files. Always test new versions locally before changing the image tag in production.

---

## Known Issues

1. **Pi-hole resets interface settings (`DNSMASQ_LISTENING=all`)**
   - Solved by correctly mapping volumes and explicitly setting the env var
   - See: `TROUBLESHOOTING.md`

2. **No HTTPS for Admin UI**
   - By design. Admin interface runs over HTTP only on the local network.
   - Can be reverse-proxied behind Nginx with TLS if needed.

3. **Pi-hole listens on all interfaces by default**
   - Restrict to `eth0` in the web interface DNS settings or firewall at the network level

---

## Future Enhancements / TODOs

- [ ] Add `.env.example` to simplify reuse
- [ ] Add optional HTTPS reverse proxy support (Nginx or Caddy)
- [ ] Backup script for adlists, `setupVars.conf`, and query logs
- [ ] Create shell/cron job to clean old logs
- [ ] Add support for Wi-Fi setup branch (for Pi 3 Model B+ with static IP)
- [ ] Include a `Makefile` or wrapper script for first-time setup

---

## Security Considerations

- The admin interface is LAN-only (HTTP on port 80).
- Docker is granted `NET_ADMIN` capability to bind privileged ports and manage internal routing.
- SSH is restricted to public-key only authentication.
- No WAN exposure unless explicitly routed — this deployment assumes internal network trust.

---

## Notes for Contributors

This project is structured for clarity and maintainability. Contributions should meet the following guidelines:

- Test changes locally before submitting a PR.
- Avoid modifying the user-facing `README.md` unless you’re correcting a bug.
- Add troubleshooting entries for any repeatable bugs fixed in a PR.
- Don’t submit changes that increase memory/CPU overhead unnecessarily — this is built for Raspberry Pi 3.
```
