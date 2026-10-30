## Episode 02 — Turning a broken-screen laptop into a headless Linux server

**Project:** Lenovo ThinkPad L580 → Ubuntu Server → Wi-Fi → SSH → external storage  
**Goal:** Reuse existing hardware, administer it remotely from my MSI laptop, and build a practical Linux/DevOps homelab.

> **My approach:** Don't just copy commands. Understand what each command changes, check the result, and document mistakes along the way.

### 1. Getting the server online: Wi-Fi with Netplan

After installing Ubuntu Server, I checked the network interfaces:

```bash
ip a
```

The result showed:

- `enp0s31f6`: Ethernet interface, with no cable connected (`NO-CARRIER`).
- `wlp6s0`: Wi-Fi interface, initially `DOWN`.
- No local IPv4 address on the Wi-Fi interface yet.

I wanted the Lenovo in another room, so I decided to use Wi-Fi instead of Ethernet.

First I checked the Netplan configuration files and the Wi-Fi authentication package:

```bash
ls /etc/netplan/
dpkg -s wpasupplicant | grep Status
```

I found `00-installer-config.yaml` and `99-wifi.yaml`, and confirmed that `wpasupplicant` was installed.

**Example of the Wi-Fi configuration** (`/etc/netplan/99-wifi.yaml`):

```yaml
network:
  version: 2
  renderer: networkd
  wifis:
    wlp6s0:
      dhcp4: true
      optional: true
      access-points:
        "YOUR_WIFI_NAME":
          password: "YOUR_WIFI_PASSWORD"
```

The network name and password above are **placeholders**, not my actual credentials. My real configuration file stays on the server and is **not committed to GitHub**.

```bash
sudo chmod 600 /etc/netplan/99-wifi.yaml  # Restrict access to credentials
sudo netplan generate                  # Validate/generate network configuration
sudo netplan apply                     # Apply the configuration
ip -4 addr show wlp6s0                # Check the Wi-Fi IPv4 address
hostname -I                            # Show local IP addresses
```

**What I learned:** Installing a server and connecting it to the network are two different tasks. `ip a` helps diagnose interfaces; Netplan makes the network configuration persistent.

### 2. Managing Ubuntu Server remotely with SSH

During installation, I enabled **OpenSSH Server**. Once the Wi-Fi worked, I could connect from **Windows PowerShell on my MSI**:

```powershell
ssh adir@YOUR_LENOVO_IP
```

`YOUR_LENOVO_IP` must be replaced with the actual local IP address shown by `hostname -I` on the Lenovo.

Useful checks on Ubuntu:

```bash
sudo systemctl is-active ssh      # Expected: active
sudo systemctl is-enabled ssh     # Expected: enabled
sudo systemctl enable --now ssh   # Enable/start SSH if needed
```

To confirm that a terminal session is connected over SSH:

```bash
echo "$SSH_CONNECTION"
```

**Result:** I can manage the Lenovo from my MSI without using its damaged built-in display.

### 3. Keeping the server running when the lid is closed

A laptop may suspend when its lid closes. That is not ideal for a server that should remain reachable by SSH.

I created a `systemd-logind` configuration file:

```bash
sudo mkdir -p /etc/systemd/logind.conf.d
sudo nano /etc/systemd/logind.conf.d/10-lid.conf
```

The file contains:

```ini
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

I then rebooted and tested SSH again with the lid closed:

```bash
sudo reboot
```

**Mistake I made:** I initially used the wrong filename, `10-lid-conf` (without the `.conf` extension). I corrected it with:

```bash
sudo mv /etc/systemd/logind.conf.d/10-lid-conf \
  /etc/systemd/logind.conf.d/10-lid.conf
```

**What I learned:** A small filename mistake can prevent a configuration from being loaded. The `.conf` extension matters in this directory.

### 4. Checking temperatures and battery health

Because I plan to run the laptop for long periods, I wanted to monitor its hardware.

```bash
sensors
```

Some readings were around **26–35°C**, but these came from storage/system sensors and should **not** be interpreted as confirmed CPU temperatures.

I also inspected the battery:

```bash
for bat in /sys/class/power_supply/BAT*; do
  [ -d "$bat" ] || continue
  echo "=== $(basename "$bat") ==="
  for f in capacity status energy_full energy_full_design cycle_count; do
    [ -f "$bat/$f" ] && echo "$f: $(cat "$bat/$f")"
  done
done
```

At the time of testing, `BAT0` reported:

| Metric | Value |
|---|---:|
| Charge | 97% |
| Full-charge capacity | 40.22 Wh |
| Design capacity | 45.73 Wh |
| Charge cycles | 57 |
| Estimated battery health | About 88% |

Battery health estimate: `energy_full / energy_full_design × 100`. This is an approximation based on battery-reported values.

I also found that the ThinkPad exposes battery charge thresholds:

```bash
ls /sys/class/power_supply/BAT0/charge_control_*_threshold
cat /sys/class/power_supply/BAT0/charge_control_{start,end}_threshold
```

For a laptop that stays plugged in, a possible setting is to **start charging at 60% and stop at 80%**:

```bash
# Optional: only if supported and after checking the existing values
# echo 60 | sudo tee /sys/class/power_supply/BAT0/charge_control_start_threshold
# echo 80 | sudo tee /sys/class/power_supply/BAT0/charge_control_end_threshold
```

These last two commands are presented as an **optional configuration**; their persistence should be checked after reboot.

### 5. Connecting a 1 TB external hard drive

I connected my external USB hard drive, formatted as **NTFS**, and listed the storage devices:

```bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINTS,TRAN
```

The important result was:

```text
sda       931.5G                              usb
└─sda1    931.5G ntfs  [external drive label]
nvme0n1   238.5G                              nvme
├─nvme0n1p1   1G vfat  /boot/efi
└─nvme0n1p2 237.4G ext4  /
```

The USB drive was detected but **not yet mounted**. In Linux, detecting a disk is not the same as making its files available through a directory.

I created a mount point and mounted the partition **read-only** for safety:

```bash
sudo mkdir -p /mnt/disque-externe
sudo mount -o ro /dev/sda1 /mnt/disque-externe
ls -lah /mnt/disque-externe
```

The files became accessible from my MSI via SSH. The option `ro` stands for **read-only**: I can read the existing files without changing them through this mount.

### 6. Automatically mounting the USB disk at startup: `/etc/fstab`

A device name such as `/dev/sda1` can change after a reboot. I therefore used the filesystem's **UUID** instead.

```bash
sudo blkid /dev/sda1                         # Find the disk's UUID
sudo cp -a /etc/fstab /etc/fstab.bak          # Back up the mount configuration
sudo nano /etc/fstab                         # Edit mount entries
```

I added an entry following this pattern (**replace `YOUR_UUID`**):

```fstab
UUID=YOUR_UUID /mnt/disque-externe auto ro,nofail,x-systemd.device-timeout=10s 0 0
```

| Option | Meaning |
|---|---|
| `UUID=...` | Identify the correct partition even if `/dev/sdX` changes |
| `/mnt/disque-externe` | Directory where the drive becomes accessible |
| `auto` | Automatically detect the filesystem type in this field |
| `ro` | Mount read-only |
| `nofail` | Allow Ubuntu to boot if the external drive is absent |
| `x-systemd.device-timeout=10s` | Limit how long boot waits for that device |
| `0 0` | Disable legacy dump/fsck processing for this entry |

Before rebooting, I verified the configuration and tested it:

```bash
sudo findmnt --verify --verbose
cd ~
sudo umount /mnt/disque-externe
sudo systemctl daemon-reload
sudo mount /mnt/disque-externe
findmnt -no SOURCE,FSTYPE,OPTIONS /mnt/disque-externe
```

In the output, I checked that the options included `ro`.

**Unplug/replug workflow without restarting Ubuntu:**

```bash
# BEFORE unplugging (wait for success before removing the USB cable):
cd ~
sudo umount /mnt/disque-externe

# AFTER plugging it in again:
sudo mount /mnt/disque-externe
ls /mnt/disque-externe
```

**Important:** This `fstab` setup mounts the disk at boot, but does **not** automatically remount it merely because the USB cable was reconnected. I chose a manual remount after reconnection for simplicity and predictability.

### 7. Challenges and lessons learned

| Challenge | What I did | What I learned |
|---|---|---|
| Damaged laptop screen | Used an external HDMI monitor for installation | A broken display does not necessarily mean an unusable computer |
| Ubuntu installed without Wi-Fi connectivity | Investigated `ip a`, `wpasupplicant`, and Netplan | Diagnose networking one layer at a time |
| Wanted to manage the server without a display | Enabled SSH and connected from Windows | Remote administration is essential for a headless server |
| Laptop suspended when closed | Configured `systemd-logind` | Power-management settings matter on repurposed laptops |
| Wrong `10-lid-conf` filename | Renamed it to `10-lid.conf` | Configuration filenames and syntax matter |
| External NTFS disk was visible but inaccessible | Mounted its partition under `/mnt/disque-externe` | Discovering storage and mounting it are different operations |
| Wanted the USB disk available after reboot | Added an `fstab` entry using UUID and `nofail` | Persistent configuration needs careful validation |

### 8. Where the project stands

**Working now:**

- [x] Ubuntu Server installed instead of Windows
- [x] Wi-Fi networking configured
- [x] Remote administration over SSH from my MSI
- [x] Laptop works with its lid closed
- [x] Battery and thermal information inspected
- [x] External 1 TB NTFS disk accessible read-only
- [x] External disk mounted automatically at boot

**Next experiments (not deployed yet):**

- [ ] Install Docker Engine and Docker Compose
- [ ] Deploy Jellyfin to index my existing media library
- [ ] Learn container networking, volumes and permissions
- [ ] Establish a proper backup strategy (a single USB disk is not a backup)
- [ ] Document troubleshooting and automation with Git

> **Takeaway:** The most useful part of this project hasn't been installing software; it has been learning how to diagnose problems, understand commands, verify outcomes, and document the process.

### References

- [Ubuntu Server documentation](https://documentation.ubuntu.com/server/)
- [Netplan examples](https://canonical-netplan.readthedocs-hosted.com/en/latest/examples/)
- [systemd mount units and `/etc/fstab` options](https://www.freedesktop.org/software/systemd/man/latest/systemd.mount.html)
- [Linux power-supply class](https://docs.kernel.org/power/power_supply_class.html)
