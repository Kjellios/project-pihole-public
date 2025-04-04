# Complete Step-by-Step Pi-hole Setup on Raspberry Pi 3 with Docker

This guide documents every step required to set up Pi-hole inside Docker on a Raspberry Pi 3 running Raspberry Pi OS Lite (32-bit). It includes all necessary configurations, commands, and considerations.

---

## Step 1: Install Raspberry Pi OS Lite on the SD Card

### 1. Download Raspberry Pi OS Lite (32-bit)
- Obtain the image from the official Raspberry Pi website.
- Choose **"Raspberry Pi OS Lite (Legacy, 32-bit)"** for best compatibility with the Raspberry Pi 3.

### 2. Flash the OS to an SD Card
- Use Raspberry Pi Imager to write the image to a microSD card.
- Select Raspberry Pi OS Lite (Legacy, 32-bit) as the operating system.
- Choose the correct SD card.

### 3. Configure OS Customization Settings (Before Writing the Image)
- Set Hostname: `pihole.local`
- Set Username & Password:
  - Username: `user`
  - Password: `[entered password]`
- Enable SSH:
  - Allowed public-key authentication only.
  - Copied SSH public key for secure authentication.
- Set Locale & Keyboard Settings:
  - Time zone: `America/Chicago`
  - Keyboard layout: `US`
- Checked "Eject media when finished".
- Did NOT configure Wi-Fi (used Ethernet for stability).

### 4. Flash OS to SD Card and Boot Raspberry Pi
- Insert the microSD card into the Raspberry Pi 3.
- Connect the Raspberry Pi to power and boot it up.
- Use a wired Ethernet connection for stability.

---

## Step 2: Connect to the Raspberry Pi via SSH

### 1. Find the Raspberry Pi’s IP Address

```sh
ping pihole.local
```

- Alternatively, check the device’s IP address in the router’s admin panel.

### 2. Connect via SSH

```sh
ssh user@192.168.1.2
```

- Confirm successful SSH connection.

---

## Step 3: Update and Prepare the System

### 1. Update the System

```sh
sudo apt update && sudo apt upgrade -y
```

### 2. Install Required Dependencies

```sh
sudo apt install -y curl git
```

- `curl` is used for downloading external scripts.
- `git` is useful for future installations (not required for Pi-hole itself).

---

## Step 4: Install Docker

### 1. Install Docker Using the Official Script

```sh
curl -fsSL https://get.docker.com | sh
```

### 2. Add the User to the Docker Group

```sh
sudo usermod -aG docker $USER
```

### 3. Apply Group Changes Without Rebooting

```sh
newgrp docker
```

### 4. Enable and Start Docker Service

```sh
sudo systemctl enable docker
sudo systemctl start docker
```

### 5. Verify Docker Installation

```sh
docker --version
docker ps
```

---

## Step 5: Install Docker Compose

### 1. Install Docker Compose

```sh
sudo apt install -y docker-compose
```

### 2. Verify Installation

```sh
docker-compose --version
```

- Output should confirm that `docker-compose` is installed (v1.29.2 or similar).

---

## Step 6: Create the docker-compose.yml File

### 1. Create Pi-hole Directory and Config Folders

```sh
mkdir -p ~/pihole/etc-pihole ~/pihole/etc-dnsmasq
cd ~/pihole
```

### 2. Create the Docker Compose File

```sh
nano docker-compose.yml
```

### 3. Add the Following Configuration

```yaml
version: "3"
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    restart: unless-stopped
    network_mode: "host"
    environment:
      TZ: "America/Chicago" # Set container timezone
      WEBPASSWORD: "${PIHOLE_WEBPASSWORD:-changeme}" # Web UI password (set in .env or default to 'changeme')
      DNSMASQ_LISTENING: "local" # Restrict DNS to local interfaces
      PIHOLE_DNS_1: "1.1.1.1" # Cloudflare DNS — fast, privacy-respecting
      PIHOLE_DNS_2: "9.9.9.9" # Quad9 DNS — security focused, blocks malicious domains
      DNSMASQ_USER: "root" # Required for host networking mode to work with Pi-hole
    volumes:
      - "./etc-pihole:/etc/pihole" # Mount for persistent Pi-hole config
      - "./etc-dnsmasq:/etc/dnsmasq.d" # Mount for custom DNSMasq overrides
    cap_add:
      - NET_ADMIN # Allow container to modify network interfaces
```

- Save and exit with CTRL + X, then Y, then ENTER.

### 4. Set Permissions for Mounted Folders

```sh
sudo chown -R 1000:1000 ./etc-pihole ./etc-dnsmasq
sudo chmod -R 755 ./etc-pihole ./etc-dnsmasq
```

---

## Step 7: Start Pi-hole in Docker

### 1. Start the Pi-hole Container

```sh
docker-compose up -d
```

### 2. Verify the Container is Running

```sh
docker ps
```

---

## Step 8: Configure Network to Use Pi-hole

### 1. Find Raspberry Pi’s IP Address

```sh
hostname -I
```

- Example output: `192.168.1.2 172.17.0.1`
- Use the local network IP (e.g. `192.168.1.2`) as the DNS server.

### 2. Configure Asus Router DNS Settings

- WAN Settings:
  - Go to WAN → Internet Connection
  - Under WAN DNS Settings, set DNS Server 1 to: `192.168.1.2`
- LAN Settings:
  - Go to LAN → DHCP Server
  - Set DNS Server 1 to: `192.168.1.2`
  - Set Advertise Router’s IP in addition to User Specified DNS to **No**
  - Enable Manual Assignment and assign `192.168.1.2` to the Pi-hole device
- Reboot Router to apply settings

---

## Step 9: Access Pi-hole Web Interface

### 1. Open the Web Interface

- Go to: `http://192.168.1.2/admin`
- Log in using the password set in `docker-compose.yml`

### 2. Adjust Privacy Settings

- Go to **Settings → Privacy**
- Set DNS Resolver Privacy Level to:
  - **High** (Domains Display & Store All Domains is Hidden)

### 3. Adjust DNS Settings

- Go to **Settings → DNS**
- Change Interface Settings to:
  - Listen only on interface `eth0` *(or leave as "all" depending on setup)*

---

## Pi-hole Setup Successfully Completed

Pi-hole is now fully operational inside Docker on a Raspberry Pi 3 and acting as a network-wide DNS sinkhole with persistent configuration and secure access.

---

## Troubleshooting

If you encounter issues, check the [Troubleshooting Guide](TROUBLESHOOTING.md) (if you've created one), or use the following command to view logs:

```sh
docker logs pihole
```
