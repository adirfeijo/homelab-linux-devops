# Homelab : My Linux & DevOps Journey
My hands-on Linux and DevOps journey: repurposing laptops, self-hosting services, and documenting troubleshooting and automation.

## Episode 01 — The beginnings 🐧

## Why I Started
Over time, I realized that I had many online subscriptions, such as : Netflix (8€), Disney+ (11€), Amazon Prime Video (7€), Spotify (12€) and Google One (30€/ year mostly for my Google Photos).

The total amount I was paying each month was a lot : 40,5 €/month

And the annual amount : 486 €

The worst part, is knowing that the movie that I wanted to watch isn't available on the 3 streaming platforms that I pay for, or maybe it's not available anymore.
Try explaining this to some friends you invited home for a movie night.

What about this message : "Your storage is 91% full. Once you run out of space, you won't be able to send and receive emails, back up new photos... Upgrade your Google One plan to get more storage"

Thats when I thought : something needs to change!

I need to own my own movies (all of them), have all my music available for free for all my devices and have more control over my data.

And since I had some spare laptops, I knew that it would be possible.
That's where I got into the rabbit hole of homelabbing, searching on YouTube, reading articles...
Then I took action.

I'll be explaining everything in this repository and documenting my journey.

## Project Goals
The idea is to replace my current subscriptions with free open-source ones.

Google Photos --> Immich

Google One --> Nextcloud

Netflix, Disney+ and Prime Video and Spotify --> Jellyfin 

## Hardware
I started out with my wifes' old Sony Vaio Laptop from 2010. Then moved on to my other laptop : a friends old Lenovo L580.
Here are the specs.

Lenovo ThinkPad L580

Processor : Intel(R) Core(TM) i5-8350U CPU @ 1.70GHz

RAM : 8GB

Storage : 256GB

## Installation process

<img width="4080" height="3072" alt="l580 before" src="https://github.com/user-attachments/assets/5984810b-c06a-40e3-bf78-19d0ca2b75c5" />

As the image shows, I didn't have the easiest start. 

The laptop's screen is partially broken (upper left corner) and it has windows with an account and password I don't have access to.
The installation process wasn't easy.

1st step : 
Download ubuntu server iso.(https://ubuntu.com/download/server)
Make your USB a bootable stick with the iso and Rufus. (https://rufus.ie/en/)

2nd step : booting up with USB Stick 

1 - Turn off the laptop

2 - Connect the Lenovo L580 to an external monitor through HDMI

3 - Press power on button

4 - When the screen turns on, press F11 button several times. This enables to enter the BIOS to change the boot order.
By default, the boot order is Windows then the USB stick. We have to choose to boot with the USB stick thanks to the arrows and enter button

5 - Accept or refuse install options
<img width="1125" height="362" alt="image" src="https://github.com/user-attachments/assets/a74f967f-f1ed-4817-a59e-68958d652b1b" />
Use entire disk (the option I chose since I only want to use Linux and remove the Windows OS)

6 - Install OpenSSH server. This will be very useful to connect to this server remotely with another computer
<img width="1318" height="992" alt="image" src="https://github.com/user-attachments/assets/78c8e2ff-be23-48a4-844b-41bd72e5ccef" />


Since my L580 computer wasn't connected to the router with e-thernet cable (which is recommended) I had to configure my laptop through wifi, which wasn't built-in. 

## Episode 02 — From a broken screen to a headless Linux server 🐧

I successfully turned a Lenovo ThinkPad L580 with a damaged screen into a working Ubuntu Server, remotely accessible from my MSI laptop.

### What I've achieved

- ✅ Installed Ubuntu Server and removed Windows
- ✅ Configured Wi-Fi using Netplan
- ✅ Enabled SSH for remote administration
- ✅ Configured the laptop to run with its lid closed
- ✅ Monitored battery health and system temperatures
- ✅ Mounted a 1 TB external NTFS hard drive
- ✅ Configured persistent storage using `/etc/fstab`
- ✅ Documented troubleshooting steps and lessons learned

**One of my biggest lessons:** Linux administration is not just about running commands. It's about understanding problems, testing solutions, and learning from mistakes.

📖 **[Read my full technical walkthrough — Commands, troubleshooting and lessons learned](docs/episode-02-lenovo-l580.md)**

### What's next?

My next goal is to install Docker and Docker Compose, followed by Jellyfin to start building my own self-hosted media server.

## Episode 03 — Secure remote access with Tailscale and SSH keys 🔐

After making my Lenovo homelab accessible over SSH on my home network, I wanted to reach it securely from elsewhere without exposing SSH directly to the public internet.

### What I achieved

- ✅ Connected the Ubuntu Server and my Windows MSI through a private Tailscale network.
- ✅ Solved a Tailscale authentication error (`404`) by generating a fresh login URL.
- ✅ Created a passphrase-protected Ed25519 SSH key on Windows.
- ✅ Installed the public key on Ubuntu and verified key-based SSH login.
- ✅ Configured OpenSSH to reject password authentication and tested a rejected password-only attempt.
- 🔄 Next: verify connectivity from a separate network and review Tailscale access policies.

**What I learned:** Secure access requires more than a working login. I need to test the allowed path, test the denied path, and preserve a recovery method.


📖 **[Read Episode 03 — Commands, troubleshooting, security checks and lessons learned](docs/episode-03-secure-remote-access.md)**

**Next:** Docker and Docker Compose, followed by Jellyfin to serve my existing media library.


## Episode 04 — My first self-hosted media server: Docker + Jellyfin 🍿🐳

My Lenovo homelab now runs **Jellyfin in Docker**, with movies and TV shows read from an external **1 TB NTFS drive**. The media stays **read-only**, while Jellyfin's configuration is stored on the internal SSD.

### What I accomplished

- ✅ Installed Docker Engine and Docker Compose using Docker's official APT repository.
- ✅ Successfully ran my first container: `hello-world`.
- ✅ Deployed Jellyfin using a `compose.yaml` file and persistent configuration folders.
- ✅ Kept the Jellyfin container's web port bound to `127.0.0.1:8096`.
- ✅ Configured **Tailscale Serve** for private HTTPS access without opening router ports.
- ✅ Accessed Jellyfin from my **MSI laptop** and **Pixel 9**, including over **4G/5G**.
- ✅ Confirmed **Direct Play** during movie playback on the Pixel 9.
- ✅ Backed up Jellyfin's configuration and copied the archive to my MSI.
- ✅ Verified the media mount, Docker, Jellyfin and Tailscale Serve after rebooting Ubuntu.

### A problem I encountered

Only **61 movies** appeared at first. I thought files were missing, but Jellyfin had not finished scanning the library. Once the scan completed, all my movies and TV series appeared.

**What I learned:** a service is not fully validated just because its container is running. I tested storage, networking, actual playback, backups and reboot behavior.

📖 **[Read Episode 04: Docker, Jellyfin & Tailscale Serve — commands, configuration and lessons learned](docs/episode-04-docker-jellyfin.md)**

**Next:** improve USB-mount readiness and automated backups, then explore Immich and Nextcloud.

## Episode 05 — Migrating my media library to a 2 TB ext4 drive

**Status:** ✅ Done (Nov 10, 2026), full checksum pass still to do  
**Journal:** [episode-05-media-disk-migration.md](episode-05-media-disk-migration.md)

I moved my Jellyfin library from an almost-full NTFS USB disk to a 2 TB ext4 drive, without reconfiguring Jellyfin and while keeping the old disk untouched as a safety net.

**Results**
- 74,483 files (about 867 GB) copied with `rsync` inside `tmux` in 3 h 05, exit code `0`
- File names, sizes, dates and counts verified identical; SMART counters still at 0 after the copy
- Jellyfin switched with a one-line Compose change; survived a reboot; plays on a Pixel 9 over 5G and on a PC
- Recovered from an unexpected USB drop of the dock (cause still unknown): `mount -a` by UUID, container restart, read-only `fsck`

**What I practised**  
Disk identification and SMART checks through a USB bridge · ext4 sizing (`-T largefile`) · `/etc/fstab` with UUID and `nofail` · resumable copies with rsync and tmux · layered verification · incident analysis with `journalctl`

**Known limitations (documented in the journal)**  
Full checksum comparison not run yet · Jellyfin does not wait for the disk at boot · a single copy is not a backup
