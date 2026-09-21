# My Home Lab — ARM64 Self-Hosted Server on a TV Box

I had an old Android TV box sitting around doing nothing. Turns out it makes a surprisingly decent low-power home server. This repo documents what I've set up on it and how.

The hardware is constrained, the budget is minimal, but it runs everything I need.

---

## Hardware

**Device:** Fiberhome HG680P (repurposed Android TV box)  
**Architecture:** ARM64  
**RAM:** 2GB  
**Storage:** 8GB eMMC + 256GB external HDD  

<img src="img/STB-HG680P.png" width="300">

Not exactly a rack server, but it handles Docker, Nextcloud, and a Cloudflare tunnel without breaking a sweat.

---

## Network Layout

<img src="img/topology.png" width="300">

---

## OS — Armbian

Replaced stock Android with **Armbian OS (v25.05.0)** built for Amlogic S905X. Flashed it using Balena Etcher.

Getting it to boot was a bit of a journey — the STB defaults to booting Android and ignores external storage unless you force it. A `reboot update` command from Android's terminal emulator does the trick, which tells the bootloader to check external storage on the next boot.

> If you're trying to do the same thing and hit a `DDR_ENC.USB` error or end up with a bricked device, I wrote a separate guide for that: [amlogic-s905x-unbrick](https://github.com/WanzzNeverSleep/docs/amlogic-s905x-DDR-ENC-error.md)

---

## Containers — Docker + CasaOS

I use Docker for everything and CasaOS as a dashboard to manage it without typing `docker ps` every five minutes.

```bash
# Docker
sudo apt-get update && sudo apt-get install docker-ce

# CasaOS
curl -fsSL get.casaos.io/install.sh | sudo bash
```

---

## Remote Access — Cloudflare Tunnel

I wanted to reach my services from outside the house without opening ports on my router. Cloudflare Tunnel handles this cleanly — it's an outbound-only connection, so nothing is directly exposed to the internet.

I bought a domain (`wanzz.my.id`) and routed everything through Cloudflare's zero trust network.

```bash
# Install cloudflared
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main' | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt-get update && sudo apt-get install cloudflared

# Set up the tunnel
sudo cloudflared service install <CLOUDFLARE_TOKEN>
sudo systemctl start cloudflared
```

---

## Self-Hosted Cloud — Nextcloud + MariaDB + Redis

My main use case. I don't love having my files on Google Drive or iCloud, so I run Nextcloud on the HDD for storage.

The stack is Nextcloud + MariaDB 11 + Redis. MariaDB because it's lighter than MySQL on ARM, Redis to cache file metadata so the UI doesn't crawl.

### Directory Setup

```bash
mkdir -p ~/docker/nextcloud/{nextcloud,mariadb,redis}
cd ~/docker/nextcloud

# External storage for Nextcloud data (owned by www-data, UID 33)
sudo mkdir -p /mnt/storage/nextcloud-data
sudo chown -R 33:33 /mnt/storage/nextcloud-data
```

### compose.yml

```yaml
services:
  db:
    image: mariadb:11
    container_name: nextcloud-db
    restart: unless-stopped
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    environment:
      MYSQL_ROOT_PASSWORD: <MYSQL_ROOT_PASSWORD>
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: <MYSQL_PASSWORD>
      MYSQL_INITDB_SKIP_TZINFO: "1"
    volumes:
      - ./mariadb:/var/lib/mysql

  redis:
    image: redis:7-alpine
    container_name: nextcloud-redis
    restart: unless-stopped

  app:
    image: nextcloud:latest
    container_name: nextcloud
    restart: unless-stopped
    ports:
      - "8090:80"
    depends_on:
      - db
      - redis
    environment:
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: <MYSQL_PASSWORD>
      MYSQL_HOST: db
      REDIS_HOST: redis
    volumes:
      - ./nextcloud:/var/www/html
      - /mnt/storage/nextcloud-data:/var/www/html/data
```

```bash
docker compose up -d
```

### Trusted Domains

Nextcloud needs to explicitly trust the domain you're accessing it from. Add it to `config.php`:

```bash
nano ~/docker/nextcloud/nextcloud/config/config.php
```

```php
'trusted_domains' =>
  array (
    0 => '192.168.1.2:8090',
    1 => 'cld.wanzz.my.id',
  ),
```

### Result

Both services are now accessible from anywhere through Cloudflare Tunnel:

- **CasaOS:** `casa.wanzz.my.id`
- **Nextcloud:** `cld.wanzz.my.id`

---

## What's Next

- [x] Repurpose Android TV box as a Linux server
- [x] Set up Nextcloud for personal cloud storage
- [x] Secure remote access via Cloudflare Tunnel
- [ ] Network-wide ad blocking (AdGuard Home or Pi-hole)
- [ ] VPN (Wireguard)
- [ ] Automated backups

---

## Related

- [amlogic-s905x-unbrick](https://github.com/WanzzNeverSleep/docs/amlogic-s905x-DDR-ENC-error.md) — Recovery guide for bricked HG680P and other Amlogic S905X devices
