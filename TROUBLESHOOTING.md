# Troubleshooting Guide

This guide outlines solutions for known issues when running **Pi-hole in Docker on a Raspberry Pi**.

---

## Issue: Pi-hole Interface Settings Reset to "Permit All Origins" After Restart

### Problem

After restarting the Pi-hole container or rebooting the Raspberry Pi, the "Interface Settings" in the Pi-hole web UI reset to "Permit All Origins" instead of remaining set to "Allow Only Local Requests."

### Symptoms

- The setting reverts to "Permit All Origins" after restarting Pi-hole.
- Checking the configuration file inside the container shows:

  ```sh
  docker exec -it pihole cat /etc/pihole/setupVars.conf | grep DNSMASQ_LISTENING
  ```

  Output:

  ```
  DNSMASQ_LISTENING=all
  ```

- Manually editing `setupVars.conf` does not persist after restarting the container.

### Root Cause

Pi-hole resets the setting due to:

1. Incorrect volume mapping, causing `setupVars.conf` to reset on restart.
2. Pi-hole applying default settings on startup, overwriting manual changes.
3. The setting not being enforced in `docker-compose.yml`, allowing Pi-hole to override it.

---

## Solution

To ensure the setting persists, update the configuration and explicitly define it in `docker-compose.yml`.

---

### Step 1: Verify Pi-hole’s Configuration Persistence

1. Check the local configuration file:

   ```sh
   cat ~/pihole/etc-pihole/setupVars.conf | grep DNSMASQ_LISTENING
   ```

   If the output is:

   ```
   DNSMASQ_LISTENING=all
   ```

   Pi-hole has reset the file.

2. Check inside the running container:

   ```sh
   docker exec -it pihole cat /etc/pihole/setupVars.conf | grep DNSMASQ_LISTENING
   ```

---

### Step 2: Manually Modify `setupVars.conf`

1. Stop the Pi-hole container:

   ```sh
   docker-compose down
   ```

2. Edit `setupVars.conf`:

   ```sh
   nano ~/pihole/etc-pihole/setupVars.conf
   ```

   Change:

   ```
   DNSMASQ_LISTENING=all
   ```

   To:

   ```
   DNSMASQ_LISTENING=local
   ```

3. Verify the change:

   ```sh
   cat ~/pihole/etc-pihole/setupVars.conf | grep DNSMASQ_LISTENING
   ```

---

### Step 3: Ensure Correct Volume Mapping in `docker-compose.yml`

1. Open `docker-compose.yml`:

   ```sh
   nano ~/pihole/docker-compose.yml
   ```

2. Ensure the `volumes:` section is correctly set:

   ```yaml
   volumes:
     - './etc-pihole:/etc/pihole'
     - './etc-dnsmasq:/etc/dnsmasq.d'
   ```

3. Fix ownership and permissions:

   ```sh
   sudo chown -R 1000:1000 ~/pihole/etc-pihole ~/pihole/etc-dnsmasq
   sudo chmod -R 755 ~/pihole/etc-pihole ~/pihole/etc-dnsmasq
   ```

---

### Step 4: Explicitly Set DNSMASQ_LISTENING in docker-compose.yml

Edit the `environment:` section in `docker-compose.yml`:

```yaml
environment:
  - TZ=America/Chicago
  - WEBPASSWORD=${PIHOLE_WEBPASSWORD:-changeme}
  - DNSMASQ_LISTENING=local
  - PIHOLE_DNS_1=1.1.1.1
  - PIHOLE_DNS_2=9.9.9.9
```

---

### Step 5: Restart Pi-hole

```sh
docker-compose up -d --force-recreate
```

Verify the setting:

```sh
docker exec -it pihole cat /etc/pihole/setupVars.conf | grep DNSMASQ_LISTENING
```

Expected output:

```
DNSMASQ_LISTENING=local
```

---

### Step 6: Test Persistence Across Reboots

1. Restart the container:

   ```sh
   docker restart pihole
   ```

2. Check again:

   ```sh
   docker exec -it pihole cat /etc/pihole/setupVars.conf | grep DNSMASQ_LISTENING
   ```

3. Reboot the Raspberry Pi:

   ```sh
   sudo reboot
   ```

4. After reboot, SSH back in:

   ```sh
   ssh user@192.168.1.2
   docker exec -it pihole cat /etc/pihole/setupVars.conf | grep DNSMASQ_LISTENING
   ```

Expected output:

```
DNSMASQ_LISTENING=local
```

---

## ✅ Conclusion

This solution ensures Pi-hole retains the "Allow Only Local Requests" setting across container restarts and Raspberry Pi reboots. If the setting resets again, view logs with:

```sh
docker logs pihole
```