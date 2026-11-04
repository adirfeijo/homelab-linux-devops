# Episode 04 — Building My First Self-Hosted Media Server with Docker, Jellyfin & Tailscale

> **Homelab Linux & DevOps — hands-on project log**  
> **Completed:** November 4, 2026  
> **Status:** ✅ Deployed, remotely accessible, backed up, and reboot-tested  
> **Previous episode:** [Episode 03 — Secure Remote Access](episode-03-secure-remote-access.md)

## 1. The goal

After turning a Lenovo ThinkPad with a damaged display into a headless Ubuntu Server, and securing remote administration with Tailscale and SSH keys, I wanted to deploy my **first real self-hosted application**.

The goal was simple: **stream my own movie and TV-series library from a USB hard drive**, using my MSI laptop and Pixel 9, including when away from home, **without exposing Jellyfin to the public Internet**.

I chose:

- **Docker Engine** to run applications in containers.
- **Docker Compose** to describe Jellyfin's configuration in a versionable YAML file.
- **Jellyfin** as the open-source media server.
- **Tailscale Serve** to provide private HTTPS access over my tailnet.
- **Read-only media mounts** to avoid accidental modification of my existing files.

**Result:** Jellyfin is running, movies and series appear with their metadata, a film played successfully in **Direct Play** on my Pixel 9, and remote access was verified over both Wi-Fi and **4G/5G**.

> All media in a homelab should be content you are entitled to store and access. Self-hosting does not change copyright obligations.

---

## 2. My setup

| Component | Actual setup / observation |
|---|---|
| Server | Lenovo ThinkPad laptop with a damaged display (headless) |
| Operating system | Ubuntu Server 26.04.1 LTS |
| Hostname | `lenovo-homelab` |
| Administration | SSH, with Ed25519 key authentication |
| Secure networking | Tailscale; restricted access-control grants |
| RAM | Approximately **7.1 GiB** total |
| Internal drive | Approximately **238.5 GiB** NVMe SSD |
| External drive | Approximately **931.5 GiB** NTFS USB disk |
| External mount | `/mnt/disque-externe` (read-only) |
| Docker | Docker Engine + Compose plugin; `hello-world` test passed |
| Client devices tested | MSI Windows laptop and Google Pixel 9 |

I deliberately keep **application configuration on the SSD** and **media files on the external drive**.

### Architecture

```text
MSI (Windows) ─────┐
                   │ Tailscale private network
Pixel 9 ───────────┤        + HTTPS
                   ▼
             Tailscale Serve
             on lenovo-homelab
                   │
                   │ http://127.0.0.1:8096
                   ▼
          Docker container: Jellyfin
              /config → internal SSD
              /cache  → internal SSD
              /media  → USB drive (read-only)
                   │
       /mnt/disque-externe/Media/
            ├── Films/
            └── Series/
```

The external disk was already configured in Episode 02 to mount through `/etc/fstab` using its **UUID**, with `ro` and `nofail`. No formatting or media migration was needed here.

---

## 3. Checking the host before installation

Connected from the MSI:

```powershell
ssh lenovo-homelab
```

Checked memory, free disk space, and mounted filesystems:

```bash
free -h
df -h /
lsblk -f
```

**Why these commands?**

- `free -h` shows memory and swap in human-readable units.
- `df -h /` shows the space available on the SSD holding Ubuntu.
- `lsblk -f` identifies disks, filesystems and mount points.

**What I found:** the server had ample free SSD space and memory to begin with Jellyfin. My external NTFS disk was detected and mounted at `/mnt/disque-externe`.

Before attaching a media library, I checked that its mount was truly read-only:

```bash
findmnt -no SOURCE,FSTYPE,OPTIONS /mnt/disque-externe
```

The options contained **`ro`**, which means *read-only*. This was a deliberate protection for an existing collection of files.

---

## 4. Installing Docker Engine and Docker Compose

Initially, `docker --version` returned `command not found`. I installed Docker from the **official Docker APT repository**, not from a random third-party installation package.

### 4.1 Check for conflicting packages

```bash
dpkg -l | grep -E '^ii  (docker.io|docker-compose|docker-compose-v2|docker-doc|docker-buildx|podman-docker|containerd|runc) '
```

This check helps identify packages that may conflict with the official Docker packages. If installed packages appear, investigate them rather than blindly removing them.

### 4.2 Install prerequisites and the repository signing key

```bash
sudo apt update
sudo apt install -y ca-certificates curl

sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

- `curl` downloads the official signing key over HTTPS.
- `/etc/apt/keyrings/` stores repository keys.
- `chmod a+r` lets APT read the key.

### 4.3 Add Docker's APT repository

```bash
sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

This tells APT where to download signed Docker packages for the installed Ubuntu release.

### 4.4 Install Docker

```bash
sudo apt install -y \
  docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

### 4.5 Verify with the first container

```bash
sudo systemctl is-active docker
sudo docker --version
sudo docker compose version
sudo docker run --rm hello-world
```

The final command returned **`Hello from Docker!`** ✅

**What this test proves:** the Docker daemon can download an image and create, execute and remove a container. The `--rm` flag removes the stopped test container; it does not remove the downloaded image.

I continued using `sudo docker` rather than adding my normal user to the `docker` group, since access to Docker's daemon can effectively grant root-level privileges.

---

## 5. Preparing Jellyfin's persistent folders

On the Lenovo:

```bash
mkdir -p ~/homelab/jellyfin/config
mkdir -p ~/homelab/jellyfin/cache
cd ~/homelab/jellyfin
```

These folders live on the **internal SSD**. The Jellyfin database, settings and artwork should survive container recreation.

I also recorded the numeric Linux user/group IDs in a local `.env` file:

```bash
printf 'PUID=%s\nPGID=%s\n' "$(id -u)" "$(id -g)" > .env
```

The Compose file below uses these values to avoid running the Jellyfin application as root. The `.env` file in this setup stores numeric IDs, not a Wi-Fi or Jellyfin password, but I still treat local configuration files carefully.

### Filesystem layout

```text
/home/adir/homelab/jellyfin/
├── compose.yaml
├── .env
├── config/       # Persistent Jellyfin database and settings
└── cache/        # Regenerable cache files

/mnt/disque-externe/
└── Media/
    ├── Films/
    └── Series/
```

---

## 6. Deploying Jellyfin with Docker Compose

I created `~/homelab/jellyfin/compose.yaml`:

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    user: "${PUID}:${PGID}"
    restart: unless-stopped

    ports:
      - "127.0.0.1:8096:8096"

    volumes:
      - ./config:/config
      - ./cache:/cache
      - type: bind
        source: /mnt/disque-externe/Media
        target: /media
        read_only: true
        bind:
          create_host_path: false
```

### Key choices explained

| Setting | Why I used it |
|---|---|
| `jellyfin/jellyfin:latest` | Official Jellyfin container image; convenient initially, but future updates should be tested and ideally version-pinned. |
| `user: "${PUID}:${PGID}"` | Runs Jellyfin with a non-root user identity. |
| `restart: unless-stopped` | Asks Docker to bring Jellyfin back after a Docker/host restart, unless intentionally stopped. |
| `127.0.0.1:8096:8096` | The published web port is accessible from the Lenovo itself, not directly from other LAN devices or the public Internet. |
| `./config:/config` | Persistent configuration on the SSD. |
| `./cache:/cache` | Cache on the SSD. |
| `/mnt/disque-externe/Media → /media` | Makes the host's media collection visible inside the container. |
| `read_only: true` | Prevents writes from this container to the media bind mount. |
| `create_host_path: false` | Avoids silently creating a missing media source directory in this bind-mount configuration. |

> **Important limitation:** a read-only bind mount is *not* a replacement for backups. Also, a restart policy by itself does not guarantee the USB disk is ready before Jellyfin starts. Improving startup dependencies is a future task.

### Validate and run

```bash
cd ~/homelab/jellyfin
sudo docker compose config --quiet
sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs --tail=30
curl -I http://127.0.0.1:8096
```

The container started successfully and the local web service responded.

---

## 7. First-time setup through an SSH tunnel

Because Jellyfin only listened on `127.0.0.1` on the Lenovo, I used an **SSH local port forward** from a separate Windows PowerShell window:

```powershell
ssh -N -L 8096:127.0.0.1:8096 lenovo-homelab
```

Then, on the MSI, I opened:

```text
http://localhost:8096
```

**How the tunnel works:** my MSI's port `8096` is securely forwarded through SSH to the same local port on the Lenovo. The terminal running `ssh -N -L ...` had to remain open during initial configuration.

In the Jellyfin setup wizard, I:

1. Gave the server a recognizable name and chose French as the display/metadata language.
2. Created a dedicated Jellyfin administrator account (separate from Ubuntu/SSH).
3. Added two libraries:
   - **Movies:** `/media/Films`
   - **TV shows:** `/media/Series`
4. Disabled automatic router port forwarding / **UPnP**.
5. Disabled heavy optional media processing for the first scan, including **Trickplay** and chapter thumbnail extraction.
6. Did **not** save artwork or downloaded subtitle files into the media folders, because the USB disk is read-only.

**Why `/media/Films` and not `/mnt/disque-externe/Media/Films`?** Docker exposes the host folder `/mnt/disque-externe/Media` *inside the container* as `/media`.

### My first small scare: only 61 movies appeared

The library initially displayed **61 movies**, even though the disk contained more. Rather than changing file names or rebuilding the installation, I let the media scan complete. **All my movies and series eventually appeared**, with posters and metadata. ✅

**Lesson:** do not assume a library is incomplete while Jellyfin is still scanning it.

---

## 8. Enabling private remote access with Tailscale Serve

A permanent SSH tunnel was useful for setup but inconvenient on a phone. I wanted a **private HTTPS URL**, usable from approved devices over Wi-Fi or cellular data.

I already had Tailscale installed on the Lenovo, MSI and Pixel 9. In Episode 03, I had restricted tailnet access to specific devices and ports.

### 8.1 Tailscale access policy

I kept the existing rule that allows the **MSI → Lenovo TCP 22** for SSH.

I added permissions for **MSI → Lenovo TCP 443** and **Pixel 9 → Lenovo TCP 443** so those devices could access the HTTPS application. The broad allow-all rule was not re-enabled.

Example conceptual grants (use your actual approved device IPs, not the placeholders):

```json
{
  "grants": [
    {
      "src": ["<MSI_TAILSCALE_IP>"],
      "dst": ["<LENOVO_TAILSCALE_IP>"],
      "ip": ["tcp:22", "tcp:443"]
    },
    {
      "src": ["<PIXEL9_TAILSCALE_IP>"],
      "dst": ["<LENOVO_TAILSCALE_IP>"],
      "ip": ["tcp:443"]
    }
  ]
}
```

This is an **illustrative policy excerpt**, not a complete export of my tailnet policy. In the administration UI, I used separate device-specific rules.

### 8.2 Enable HTTPS, not Funnel

On first use, Tailscale asked to enable HTTPS certificates. The consent page also offered the optional **Tailscale Funnel** checkbox.

I enabled **HTTPS certificates** but **left Funnel disabled**.

- **Tailscale Serve:** shares my service only inside the private tailnet, subject to access rules.
- **Tailscale Funnel:** can make a service reachable publicly on the Internet; not what I wanted.

**Privacy note:** the HTTPS hostname may appear in public certificate-transparency records, although the application itself remains private.

### 8.3 Start Serve in the background

On the Lenovo:

```bash
sudo tailscale serve --bg 8096
sudo tailscale serve status
```

The result included a line like:

```text
https://lenovo-homelab.<YOUR_TAILNET>.ts.net (tailnet only)
|-- / proxy http://127.0.0.1:8096
```

`--bg` lets the Serve configuration run in the background and persist across service/host restarts. Jellyfin's port remained bound to localhost: **no router port forwarding was needed**.

I now use the private HTTPS URL on clients connected to my tailnet. I can close the temporary SSH forwarding window.

---

## 9. Real-world tests and results

I tested actual end-user access, not just whether a container was `Up`.

| Test | Result |
|---|---|
| Docker's `hello-world` container | ✅ Passed |
| Jellyfin web interface from MSI via local SSH tunnel | ✅ Passed |
| Jellyfin from MSI via private Tailscale HTTPS | ✅ Passed |
| Jellyfin from Pixel 9 using Wi-Fi | ✅ Passed |
| Jellyfin from Pixel 9 using 4G/5G | ✅ Passed |
| Start and watch a movie on Pixel 9 | ✅ Passed |
| Jellyfin playback mode on tested movie | ✅ **Direct Play** |
| Host reboot: reconnect by SSH/Tailscale | ✅ Passed |
| Host reboot: external NTFS drive remounted | ✅ Passed |
| Host reboot: Docker, Jellyfin and Serve recovered | ✅ Passed |

### Why Direct Play matters

In the Jellyfin dashboard, the Pixel 9 session showed **Direct Play**. The server was streaming the file without re-encoding its video. That usually reduces CPU use and preserves the file's original encoded quality.

This result only applies to the specific movie/client combination I tested. Other formats, subtitle types or clients might still trigger transcoding.

**Network detail:** Jellyfin showed a Docker-side address resembling `172.18.0.1` for the client in its dashboard. With Tailscale Serve forwarding to a Docker container, this can be a proxy/network-bridge address rather than the phone's actual IP.

---

## 10. Backing up Jellyfin and copying the backup off the server

Once my library worked, I created a **consistent offline backup** of Jellyfin's configuration.

First, I checked its size:

```bash
du -sh ~/homelab/jellyfin/config
```

It was approximately **1.8 GiB**.

### 10.1 Stop the container to avoid copying an actively changing database

```bash
cd ~/homelab/jellyfin
sudo docker compose stop
mkdir -p ~/homelab-backups
```

### 10.2 Create the archive

```bash
tar -czf ~/homelab-backups/jellyfin-$(date +%Y%m%d-%H%M%S).tar.gz \
  config compose.yaml .env
```

The archive produced during this project was approximately **1.7 GiB**. It contains the Jellyfin configuration and Compose deployment files; **it does not contain the movies and series** on the external USB drive. The separate `cache` directory was intentionally not included.

### 10.3 Restart Jellyfin

```bash
sudo docker compose start
```

### 10.4 Validate the archive

Example using the actual filename created during this session:

```bash
tar -tzf ~/homelab-backups/jellyfin-20261104-135644.tar.gz > /dev/null \
  && echo "Backup archive readable"
```

This check **passed**. It validates that the compressed archive can be read, but **does not prove a full restore**.

I restricted local access to the archive:

```bash
chmod 700 ~/homelab-backups
chmod 600 ~/homelab-backups/jellyfin-20261104-135644.tar.gz
```

### 10.5 Download it to the MSI via SCP

In **Windows PowerShell**, not inside a Linux SSH session:

```powershell
scp lenovo-homelab:/home/adir/homelab-backups/jellyfin-20261104-135644.tar.gz "$env:USERPROFILE\Downloads\"
```

The file successfully arrived on the MSI. This gives me an **off-host copy** in addition to the original on the Lenovo.

> The backup contains potentially sensitive application configuration. Store the off-host copy securely, ideally encrypted. A second independent copy and a tested restore procedure would improve resilience further.

---

## 11. Verifying automatic restart after a full reboot

Before rebooting, I checked the services:

```bash
sudo systemctl is-enabled docker tailscaled ssh
systemctl is-enabled ssh.socket
systemctl is-active ssh.socket
sudo docker inspect jellyfin --format '{{.HostConfig.RestartPolicy.Name}}'
```

**Interesting detail:** `ssh.service` showed **`disabled`**, but `ssh.socket` was **`enabled` and `active`**. This is normal for my Ubuntu installation: socket activation can start SSH when a connection arrives.

The Jellyfin container's restart policy was `unless-stopped`.

I rebooted:

```bash
sudo reboot
```

After reconnecting from the MSI with `ssh lenovo-homelab`, I checked:

```bash
# Check the USB drive was mounted, read-only
findmnt -no SOURCE,FSTYPE,OPTIONS /mnt/disque-externe

# Check Jellyfin is running
cd ~/homelab/jellyfin
sudo docker compose ps

# Check the private HTTPS proxy is still configured
sudo tailscale serve status
```

**All three checks passed.** I also successfully reached Jellyfin again. This confirmed that the system could recover after a normal reboot without reconnecting a monitor.

### A reliability improvement still on my roadmap

This successful reboot does **not** guarantee Jellyfin will always wait for the USB disk if the drive is slow to appear or disconnected. I plan to add a stronger startup dependency/check that confirms the real filesystem is mounted **before** Jellyfin starts.

Until then, if the disk is missing, I avoid running a media-library scan and stop Jellyfin until storage is restored.

---

## 12. Useful administration commands

```bash
# Active containers
sudo docker ps

# Jellyfin service status
cd ~/homelab/jellyfin && sudo docker compose ps

# Recent application logs
cd ~/homelab/jellyfin && sudo docker compose logs --tail=50

# Follow the logs in real time
cd ~/homelab/jellyfin && sudo docker compose logs -f

# Restart just Jellyfin
cd ~/homelab/jellyfin && sudo docker compose restart

# Check actual storage mount and read-only status
findmnt -no SOURCE,FSTYPE,OPTIONS /mnt/disque-externe

# Check Tailscale Serve
sudo tailscale serve status

# Check the local web endpoint
curl -I http://127.0.0.1:8096

# See Docker disk usage
sudo docker system df
```

**Caution:** do not blindly run destructive cleanup commands (`docker system prune -a`, `docker volume prune`, etc.) when learning. Understand which images, containers and data are in use first.

---

## 13. Issues, troubleshooting and lessons learned

| Issue or concern | What happened / what I learned |
|---|---|
| `docker: command not found` | Docker had not yet been installed; the official repository and packages resolved it. |
| Only 61 movies appeared initially | Jellyfin was still scanning. I waited; the complete movie and TV library appeared. |
| The container only listened on `127.0.0.1:8096` | This was intentional for security. I used an SSH tunnel first, then private Tailscale Serve. |
| Consent screen offered HTTPS and Funnel | I selected **HTTPS only**, leaving **Funnel** disabled to avoid public exposure. |
| Tailscale grants were already restrictive | I allowed TCP 443 only for the approved MSI and Pixel 9 while preserving the MSI's TCP 22 SSH access. |
| Jellyfin displayed `172.18.0.1` | Likely a local Docker networking/proxy address, not the Pixel 9's Internet IP. |
| `ssh.service` was disabled | `ssh.socket` was enabled and active; my SSH access worked after reboot. |
| External disk could be temporarily absent | A bind mount and `unless-stopped` alone are not a complete storage-readiness guarantee; a startup dependency is still planned. |
| Backup was stored on the same SSD | I transferred the tested archive to the MSI so there was also an off-host copy. |

### Key takeaways

1. **Separate applications from data.** Jellyfin's config and cache live on the internal SSD; the media is read from an external drive.
2. **Minimize exposure.** Bind Jellyfin to localhost and use a private HTTPS proxy instead of publishing it on the router.
3. **Use infrastructure as code.** `compose.yaml` captures how the service is deployed.
4. **Validate in layers.** A running Docker container, a working local HTTP endpoint, a working remote login, and actual video playback are different tests.
5. **Backups need an off-host copy.** A tarball on the same disk is not enough; checking archive readability is useful, but restoring is the ultimate test.
6. **Test reboot behavior.** A service working today does not automatically mean it starts correctly tomorrow.

---

## 14. Security notes and limitations

- SSH administration uses an Ed25519 key; password-based SSH login was disabled in Episode 03.
- Tailscale access controls restrict which devices may connect to TCP 22 and TCP 443.
- Jellyfin's backend port is published to **localhost only** (`127.0.0.1`).
- Tailscale **Serve** is enabled, but **Funnel** is not.
- The media disk is mounted `ro` on Ubuntu and the media is also bound `read_only` into Docker.
- Jellyfin has its own user accounts and authentication; Tailscale is an additional access-control layer, not a replacement.
- Docker can bypass some UFW expectations for **published container ports**. The localhost binding and tailnet policy were chosen deliberately, and should be reviewed if networking changes.
- I have not yet implemented a robust startup dependency for the USB filesystem, an automatic encrypted off-site backup, or a full backup restoration test.
- I have not yet validated multi-user streaming, every video codec, hardware transcoding or a Samsung TV client.

---

## 15. What's next?

- [x] Install Docker Engine and Docker Compose.
- [x] Deploy Jellyfin using Compose.
- [x] Use the external NTFS media disk in read-only mode.
- [x] Configure private HTTPS with Tailscale Serve.
- [x] Test MSI, Pixel 9, Wi-Fi and 4G/5G access.
- [x] Verify Direct Play on a Pixel 9 movie.
- [x] Create an archive backup and copy it to the MSI.
- [x] Verify recovery after a normal Ubuntu reboot.
- [ ] Implement a reliable USB-mount readiness check before Jellyfin starts.
- [ ] Automate backups and test a full restoration.
- [ ] Review version-pinning and a safe container-update workflow.
- [ ] Set up Immich for personal photo management and backup.
- [ ] Explore Nextcloud for private file synchronization.
- [ ] Continue publishing progress and troubleshooting notes on GitHub and LinkedIn.

---

## Official documentation

- [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [Docker Compose documentation](https://docs.docker.com/compose/)
- [Jellyfin — Docker containers](https://jellyfin.org/docs/general/installation/container/)
- [Tailscale Serve](https://tailscale.com/docs/features/tailscale-serve)
- [Tailscale — Private web server with Docker](https://tailscale.com/docs/solutions/host-private-server-docker)

*This is a learning journal based on the steps I actually performed and the tests I completed. Example IP addresses, tailnet names and secrets are intentionally omitted.*
