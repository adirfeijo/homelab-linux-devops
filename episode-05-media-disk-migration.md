# Episode 05 — Migrating My Media Library to a 2 TB ext4 Drive with rsync

> **Homelab Linux & DevOps — hands-on project log**  
> **Completed:** November 10, 2026  
> **Status:** ✅ Migrated, reboot-tested, and recovered from a USB drop; quick verification passed (full checksum pass still to do)  
> **Previous episode:** [Episode 04 — Building My First Self-Hosted Media Server with Docker, Jellyfin & Tailscale](episode-04-docker-jellyfin.md)

## 1. The goal

In Episode 04, Jellyfin streamed my films and series from a **931.5 GiB NTFS USB disk**, mounted read-only. It worked, but that disk was almost full (**89 %** according to `df`), and NTFS is a Windows filesystem that Linux reaches through a FUSE driver (the old disk showed up as `fuseblk` in `findmnt`).

The goal was to **move the whole library to a 2 TB drive formatted with a Linux-native filesystem (ext4)**, prove that nothing was lost, and switch Jellyfin over by **changing a single line** in `compose.yaml` — while keeping the old disk untouched as a safety net.

I chose:

- **ext4** for the new drive (native to Linux, real permissions, no FUSE layer).
- **smartmontools** to check the drive's health *before* trusting it.
- **rsync** to copy: it can be re-run if interrupted and it never modifies the source.
- **tmux** so an SSH disconnection could not kill a three-hour copy.
- A **one-line Compose change** so Jellyfin's libraries (`/media/Films`, `/media/Series`) did not need reconfiguring.

**Result:** **74,483 files (about 867 GB)** copied in **3 h 05 min** with exit code `0`; file lists verified identical; Jellyfin now streams from the new drive — including on my Pixel 9 over **5G** — and survived a reboot.

---

## 2. My setup

| Component | Setup / observation |
|---|---|
| Server | Lenovo ThinkPad laptop (headless), Ubuntu Server 26.04.1 LTS, hostname `lenovo-homelab` |
| Internal drive | Approximately **238.5 GiB** NVMe SSD (Ubuntu, Docker configuration) |
| Old media disk (source) | Approximately **931.5 GiB** NTFS USB disk, `/mnt/disque-externe`, mounted read-only, **824 GiB used** |
| New media disk (target) | Western Digital **WD20EURX**, 2 TB, 3.5-inch SATA (WD AV-GP family, 5400 rpm) |
| Connection | USB-to-SATA dock with an **ASMedia ASM1153E** bridge chip |
| New mount point | `/mnt/stockage` |

A 3.5-inch drive does not fit inside a laptop, so it is connected through the dock.

### Before and after

```text
BEFORE                                         AFTER
Jellyfin container                             Jellyfin container
  /config → internal SSD                         /config → internal SSD
  /cache  → internal SSD                         /cache  → internal SSD
  /media  → /mnt/disque-externe/Media            /media  → /mnt/stockage/Media
            (NTFS, USB, mounted ro)                        (ext4, USB dock, bind mount ro)
```

Only the left side of the last arrow changes. Inside the container, the path is still `/media`.

---

## 3. Identifying the right disk before anything destructive

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,FSTYPE,LABEL,MOUNTPOINTS
```

Excerpt (serial numbers omitted):

```text
NAME          SIZE MODEL                FSTYPE LABEL            MOUNTPOINTS
sda         931.5G BUP Slim SL
└─sda1      931.5G                      ntfs   Disco Duro Adir  /mnt/disque-externe
sdb           1.8T ASM1153E
└─sdb1        1.8T                      ntfs   Nouveau nom
nvme0n1     238.5G SAMSUNG MZVLB256HBHQ-000L7
├─nvme0n1p1     1G                      vfat                    /boot/efi
└─nvme0n1p2 237.4G                      ext4                    /
```

What I took from it:

- The `MODEL` column of the new disk shows the **bridge chip** (`ASM1153E`), not the drive itself. The real model only appeared later, in the SMART data.
- The serial number reported by the bridge looked like a placeholder (a plain sequence of digits), so I did **not** use it to identify the disk. I used size and, later, the drive's own identity from SMART.
- Device letters (`sda`, `sdb`) are assigned in plug-in order and can change. I re-checked right before formatting, and I use **UUIDs** in `/etc/fstab`.

---

## 4. Looking inside before erasing

The target already carried an NTFS partition labelled "Nouveau nom". Before erasing anything, I mounted it **read-only**:

```bash
sudo mkdir -p /mnt/inspection
sudo mount -o ro /dev/sdb1 /mnt/inspection
df -h /mnt/inspection
ls -la /mnt/inspection
sudo umount /mnt/inspection
```

**What I found:** 159 MB used out of 1.9T, and a single folder, `System Volume Information`. Nothing to keep.

**Lesson:** `-o ro` lets you look without any risk. Formatting is irreversible, so look first.

---

## 5. Checking the drive's health through the USB bridge

```bash
sudo apt install -y smartmontools
sudo smartctl -a -d sat /dev/sdb
```

`-d sat` selects the SCSI/ATA Translation pass-through, which smartmontools lists as supported by ASMedia bridges. It worked here.

| Item | Value |
|---|---|
| Model | `WDC WD20EURX-63T0FY0` (WD AV-GP family, 5400 rpm) |
| Sector size | 512 bytes logical, **4096 bytes physical** |
| Overall health | `PASSED` |
| `Reallocated_Sector_Ct` (5) | 0 |
| `Current_Pending_Sector` (197) | 0 |
| `Offline_Uncorrectable` (198) | 0 |
| `UDMA_CRC_Error_Count` (199) | 0 (cable and dock transmit cleanly) |
| `Power_On_Hours` | 18,107 (about 2 years of cumulative use) |
| Error log | No errors logged |
| Self-test log | Short tests only |

**Caveat:** a passing SMART verdict is not a guarantee, and short self-tests do not scan the whole surface. This is one reason I keep the old disk untouched.

---

## 6. Partitioning, formatting and mounting

```bash
sudo parted /dev/sdb --script mklabel gpt mkpart stockage ext4 1MiB 100%
lsblk -o NAME,SIZE,FSTYPE,LABEL /dev/sdb
sudo mkfs.ext4 -L stockage -m 1 -T largefile /dev/sdb1
```

- **GPT, partition starting at 1 MiB:** 1 MiB is a multiple of 4096, so the partition is aligned with the drive's physical sectors.
- **`mkfs` warning:** it reported an existing NTFS filesystem and asked `Proceed anyway? (y,N)`. I had inspected the disk, so I answered `y`.
- **`-m 1`:** reserves 1 % of the space for root instead of the default 5 % — roughly 20 GB instead of 100 GB on a drive this size. This is a pure data disk.
- **`-T largefile`:** creates one inode per MiB instead of the default ratio of one per 16 KiB.
  - On a drive this size, the default would create about **122 million inodes**, i.e. roughly **31 GB** of inode tables (computed on a test image of the same size).
  - With `largefile`, `mkfs` created **1,907,840 inodes** — about 0.5 GB.
  - The trade-off: this suits large video files, not millions of tiny files.

Then the mount, by UUID:

```bash
sudo mkdir -p /mnt/stockage
UUID=$(sudo blkid -s UUID -o value /dev/sdb1); echo "$UUID"
[ -n "$UUID" ] && echo "UUID=$UUID  /mnt/stockage  ext4  defaults,nofail  0  2" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
findmnt /mnt/stockage
sudo chown adir:adir /mnt/stockage
df -h /mnt/stockage
```

The guard (`[ -n "$UUID" ] &&`) prevents writing a broken `fstab` line if the UUID lookup returns nothing. The result: `/dev/sdb1` mounted as `ext4`, 2.1M used, 1.8T available.

---

## 7. Copying the library with rsync

A dry run (`-n`) shows what rsync would do without writing anything:

```bash
rsync -rtnh --stats --exclude='System Volume Information' --exclude='$RECYCLE.BIN' \
  /mnt/disque-externe/ /mnt/stockage/ | tail -n 20
```

The real copy, inside `tmux`:

```bash
tmux new -s copie
rsync -rth --chmod=D755,F644 --info=progress2 \
  --exclude='System Volume Information' --exclude='$RECYCLE.BIN' \
  /mnt/disque-externe/ /mnt/stockage/
```

**Why these options?**

- `-r` copies recursively and `-t` keeps modification times. I did not use `-a`: NTFS has no real Unix owners or permissions, so there is nothing meaningful to preserve.
- `--chmod=D755,F644` gives directories `755` and files `644`. NTFS shows everything as `777` on Linux (`drwxrwxrwx` / `-rwxrwxrwx`).
- The two `--exclude` options skip Windows system folders.
- The **trailing slashes** matter: they copy the *contents* of the source, not the folder itself.
- `tmux` keeps the job alive if the SSH session drops (detach with `Ctrl+b` then `d`, reattach with `tmux attach -t copie`).
- rsync does not delete anything and does not modify the source. If it is interrupted, re-running the same command resumes the work.

**Result** (final progress line, then `echo $?`):

```text
866.97G 100%   74.54MB/s    3:04:51 (xfr#74483, to-chk=0/89637)
0
```

74,483 files transferred (about 867 GB, ≈ 807 GiB), 89,637 entries (files and folders) checked, exit code **`0`**.

---

## 8. Verifying the copy, layer by layer

### 8.1 Quick check: names, sizes, dates, counts

```bash
rsync -rtn --itemize-changes --exclude='System Volume Information' --exclude='$RECYCLE.BIN' \
  /mnt/disque-externe/ /mnt/stockage/

find /mnt/stockage -type f | wc -l
find /mnt/disque-externe -type f \
  -not -path '*/System Volume Information/*' -not -path '*/$RECYCLE.BIN/*' | wc -l
```

- The dry run printed **nothing**: no file differs in name, size or modification time.
- Both `find` commands returned **74,483**.
- `find` also printed `Permission denied` for `lost+found`: that folder is root-only, and empty on a new filesystem.

Note that rsync's default comparison uses **size and modification time only**. A modified byte that changes neither would go unnoticed; only a checksum comparison (`-c`) would catch it. I have not run that full pass yet (see section 14).

### 8.2 Drive health after three hours of writing

```bash
sudo smartctl -A -d sat /dev/sdb | grep -E "Reallocated_Sector|Current_Pending|Offline_Uncorrectable|UDMA_CRC|Temperature"
```

The four counters were still at **0**; the temperature was 34 °C.

### 8.3 Kernel log

```bash
sudo dmesg -T --level=err,warn | grep -v 'UFW BLOCK' | tail -n 40
```

My first attempt, `dmesg … | tail -n 20`, showed only firewall lines: UFW was logging blocked network multicast (IGMP) queries every couple of minutes, which pushed everything older out of view. **Filter the noise before concluding that there are no errors.**

The filtered output contained no read or write errors:

- Wi-Fi beacon warnings from before the copy.
- Two warnings at the moment the disk was plugged in, saying that the I/O sizes reported by the USB bridge are not multiples of 4096 bytes. They appeared before partitioning and were not followed by any read or write error.

This filter only shows warnings and errors. Together with exit code `0` and unchanged SMART counters, a silent problem is unlikely, but it is not proven.

### 8.4 Space used

```bash
df -h /mnt/disque-externe /mnt/stockage
sudo du -sh '/mnt/disque-externe/$RECYCLE.BIN' '/mnt/disque-externe/System Volume Information'
```

| Disk | Used |
|---|---|
| Old NTFS disk | 824G |
| New ext4 drive | 808G |

The 16 GiB difference is explained by the Windows Recycle Bin: `$RECYCLE.BIN` weighs **16G** and `System Volume Information` only 49K. rsync excluded both on purpose. The entries I listed in the Recycle Bin were dated 2017 and 2019. They stay on the old disk.

---

## 9. Switching Jellyfin to the new drive

First, I made sure nothing else depended on the old mount:

```bash
sudo docker compose ls
grep -rn "disque-externe" /home/adir/homelab --include='*.yaml' --include='*.yml' --include='.env' 2>/dev/null
sudo docker inspect $(sudo docker ps -q) --format '{{.Name}}: {{range .Mounts}}{{.Source}} -> {{.Destination}}; {{end}}'
```

Only Jellyfin's `/media` bind mount (line 17 of its `compose.yaml`) pointed at the old disk. The other containers use folders on the internal SSD.

Then the change, with a backup kept in `/tmp` so it cannot end up in a Git repository by accident:

```bash
cd ~/homelab/jellyfin
cp compose.yaml /tmp/compose.yaml.bak
sed -i 's#/mnt/disque-externe/Media#/mnt/stockage/Media#' compose.yaml
diff /tmp/compose.yaml.bak compose.yaml
sudo docker compose config --quiet && echo "compose OK"
sudo docker compose up -d
sudo docker inspect jellyfin --format '{{range .Mounts}}{{.Source}} -> {{.Destination}} (rw={{.RW}}); {{end}}'
```

The `diff` showed exactly one changed line:

```diff
-        source: /mnt/disque-externe/Media
+        source: /mnt/stockage/Media
```

`target: /media` and `read_only: true` were not touched, so the libraries (`/media/Films`, `/media/Series`) did not need any reconfiguration. The container was recreated in about a second, and the final `inspect` confirmed the new source:

```text
/mnt/stockage/Media -> /media (rw=false)
```

---

## 10. Tests and results

| Test | Result |
|---|---|
| rsync exit code | ✅ `0` |
| File names, sizes and dates identical (dry run) | ✅ No differences |
| File count, source vs. target | ✅ 74,483 = 74,483 |
| SMART critical counters after the copy | ✅ All at 0 |
| Kernel errors during the copy | ✅ None found (errors/warnings filter) |
| Container sees the new source, read-only | ✅ `rw=false` |
| Reboot: new drive mounted automatically | ✅ Passed |
| Reboot: old disk back, read-only | ✅ Passed |
| Reboot: Jellyfin back, health check `healthy` | ✅ Passed |
| Playback after the reboot: Pixel 9 over 5G | ✅ Passed |
| Playback after the reboot: MSI laptop | ✅ Passed |
| Playback after the USB drop, remount and container restart: MSI laptop and phone | ✅ Passed |
| Read-only `fsck` after the USB drop | ✅ Clean (five passes, no error) |

---

## 11. An incident: the dock dropped off the USB bus

Right after I unmounted and unplugged the old disk, Jellyfin refused to play anything: **"Playback failed due to a fatal player error."**

That message is generic. Forum threads trace it to wrong media paths, client codec settings, transcoding options or folder permissions, so I did not assume the cause. I checked the storage first, in read-only mode:

```bash
findmnt /mnt/stockage
lsblk
sudo journalctl -k --since "16:05" --until "16:20" --no-pager | grep -v 'UFW BLOCK'
```

`findmnt` printed nothing: `/mnt/stockage` was no longer mounted on the host. The drive had come back under a different name (`sdc` instead of `sdb`). The kernel log (server time is UTC) told the story:

| Time | Event |
|---|---|
| 16:17:22 | The 2 TB dock disconnects from USB. The kernel shuts the ext4 filesystem down. |
| 16:17:23 | The dock reconnects one second later, as a new device. |
| 16:17:26 | The old 1 TB disk is unplugged, **four seconds later**. |

So the unplug did **not** cause the drop: the dock had already disconnected. I do not know why. A loose contact on the cable, a power glitch or the dock itself are all possible, and the log does not say. As far as I know, I did not touch the dock's cable or power supply. The kernel log shows the dock was running in UAS mode, a USB transfer protocol some bridges handle badly; I am noting it as a lead, not as a proven cause.

**Why Jellyfin broke:** the host's mount had disappeared, but the container kept its own copy of the old, dead mount, so every read failed. The kernel only finished unmounting the old `sdb1` at 16:26:03, when I restarted the container.

### Recovery

```bash
sudo mount -a
cd ~/homelab/jellyfin && sudo docker compose restart
```

`mount -a` worked even though the device letter had changed, because `/etc/fstab` uses the UUID. On the new mount, the kernel logged `recovery complete`: ext4 replayed its journal.

Then a read-only check of the filesystem:

```bash
cd ~/homelab/jellyfin && sudo docker compose stop
sudo umount /mnt/stockage
sudo fsck.ext4 -fn /dev/disk/by-label/stockage
sudo mount -a && sudo docker compose start
```

`-n` forbids any modification. All five passes completed with no error: 89,648 inodes in use (the 89,637 entries rsync copied plus 11 reserved by ext4) and about 869 GB of blocks (867 GB copied plus about 2 GB of ext4 structures). Playback worked again on the MSI laptop and on my phone.

### What I could not establish

- **The cause of the drop.** The log shows what happened, not why.
- **Whether it happened earlier.** I first wrote that the systemd journal on this server was volatile. That was wrong: it is persistent, and `journalctl --list-boots` shows boots kept since 19 October. I then searched the kernel log of the previous boot (4 to 10 November, which includes the three-hour copy) for `disconnect`, `reset`, `I/O error`, `EXT4-fs` and `uas`. The dock appears at 11:59 UTC, the new filesystem is mounted at 12:15 and cleanly unmounted at 15:55 by the reboot, and nothing in between reports a USB disconnect or an I/O error. So the 16:17 drop is the only one visible since the dock was plugged in. I did not search the older boots.

---

## 12. Useful administration commands

```bash
# Drive identity and health, through the USB bridge (check the device letter first)
sudo smartctl -a -d sat /dev/sdb

# Is the media drive mounted, and how full is it?
findmnt /mnt/stockage
df -h /mnt/stockage

# Reattach to a long-running copy
tmux attach -t copie

# Full-surface self-test (the drive estimates about 4.5 hours)
sudo smartctl -t long -d sat /dev/sdb

# Which disk is which?
lsblk -o NAME,SIZE,MODEL,FSTYPE,LABEL,MOUNTPOINTS

# Kernel log for a precise time window (server time is UTC)
sudo journalctl -k --since "16:05" --until "16:20" --no-pager | grep -v 'UFW BLOCK'

# Previous boots are kept: list them, then search the previous boot for USB or disk errors
journalctl --list-boots
journalctl -k -b -1 --no-pager | grep -viE 'UFW BLOCK' | grep -iE 'disconnect|reset|I/O error|EXT4-fs|uas'

# Read-only filesystem check (unmount first; changes nothing)
sudo fsck.ext4 -fn /dev/disk/by-label/stockage
```

---

## 13. Issues, troubleshooting and lessons learned

| Issue or concern | What happened / what I learned |
|---|---|
| A 3.5-inch drive in a laptop | It does not fit; a USB-to-SATA dock solved it. |
| `lsblk` showed `ASM1153E` as the model | That is the bridge chip. The drive's real identity came from `smartctl`. |
| The bridge's serial number looked like a placeholder | I did not rely on it to identify the disk. |
| The target already had an NTFS partition | I inspected it read-only before formatting. |
| Default ext4 settings on a 2 TB drive | About 122 million inodes (≈ 31 GB). `-T largefile` created 1.9 million (≈ 0.5 GB). |
| NTFS files appeared as `777` | `--chmod=D755,F644` gave them normal permissions on ext4. |
| `dmesg` looked clean but was only firewall noise | Filter out `UFW BLOCK` before concluding anything. |
| 16 GiB less on the new drive | That was the Windows Recycle Bin, which I excluded on purpose. |
| Jellyfin: "fatal player error" after unplugging the old disk | The error is generic. The real cause was the 2 TB dock dropping off USB four seconds *before* the unplug; the container then kept a dead copy of the mount. |
| The drive's letter changed (`sdb` → `sdc`) | Mounting by UUID in `/etc/fstab` made `mount -a` work anyway. |
| I wrote that the journal was volatile | It was persistent all along; old archived journals in `/var/log/journal` proved me wrong. Check a limitation before writing it down. |

### Key takeaways

1. **Identify, inspect, then erase.** Confirm the device by more than its letter, and look at its contents read-only before formatting.
2. **Measure health first.** SMART takes seconds and tells you whether a 10-year-old drive deserves your data.
3. **Prefer resumable, verifiable tools.** rsync inside tmux survives disconnections and leaves the source untouched.
4. **Verify in layers.** Exit code, file lists, counts, drive counters, kernel log and real playback are different checks.
5. **Explain every gap.** The 16 GiB difference looked suspicious until `du` pointed at the Recycle Bin.
6. **Keep the infrastructure change small.** A one-line diff is easy to review and easy to undo: put the old line back and the old disk is still there.
7. **Check timestamps before blaming the last thing you touched.** The unplug looked guilty; the kernel log showed the other drive dropping four seconds earlier.
8. **A disk that disappears and comes back is not seen again by a running container.** Remount it on the host, then restart the container.

---

## 14. Security notes and limitations

- The new drive is mounted **read-write on the host** (I needed to write to it). Jellyfin still sees it read-only through `read_only: true`, but unlike the old NTFS mount, the protection no longer exists at host level.
- The old 1 TB disk is a **snapshot of today's library**. Media added later will not be on it. A single copy is not a backup strategy.
- I have **not** run a full checksum comparison (`rsync -rtnc --itemize-changes …`). It reads every file on both disks and takes hours. I plan to run it before reusing or erasing the old disk.
- The old disk's **Recycle Bin (about 16 GB)** was not copied.
- **Startup order is still unsolved.** One successful reboot does not guarantee that the USB drive is always ready before Jellyfin starts. In theory, if `/mnt/stockage` is not mounted, the `Media` folder does not exist and `create_host_path: false` makes the container fail to start rather than show an empty library. I have not tested that scenario.
- **A USB drop while running is not handled either.** The incident in section 11 left Jellyfin with a dead mount until I restarted the container by hand. Automatic detection and restart are still to do.
- I have **not** tested booting without the 2 TB drive attached. `nofail` lets the system boot without it, but sources disagree on whether a missing drive can still delay the boot by up to 90 seconds (the ArchWiki says so; the `systemd.mount` manual, quoted in the wiki's discussion page, says the boot does not wait).
- ext4 is not readable by Windows without extra software. Moving this drive to the MSI would need a Linux-aware tool.
- The drive has about 2 years of cumulative use and passed SMART, but it sits behind a USB dock: the dock and cable are single points of failure too.

---

## 15. What's next?

- [x] Identify and inspect the target disk before erasing it.
- [x] Check the drive's health with SMART through the USB bridge.
- [x] Partition, format as ext4 and mount by UUID with `nofail`.
- [x] Copy the library with rsync (74,483 files).
- [x] Verify names, sizes, dates and file counts.
- [x] Point Jellyfin at the new drive with a one-line Compose change.
- [x] Reboot test and playback test (Pixel 9 over 5G, MSI).
- [x] Unmount and unplug the old 1 TB disk, kept as an offline copy of today's library.
- [x] Recover from the dock dropping off the USB bus (remount, container restart, read-only `fsck`).
- [ ] Run the full `rsync -c` checksum comparison before reusing or erasing the old disk.
- [ ] Implement a reliable storage-readiness check before Jellyfin starts.
- [ ] Detect a dropped drive and restart Jellyfin automatically (health check or systemd).
- [x] Check that the systemd journal keeps history across reboots (it does: boots are kept since 19 October).
- [ ] Refresh the Jellyfin configuration backup (`compose.yaml` changed), automate backups and test a full restore.
- [ ] Continue publishing progress and troubleshooting notes on GitHub and LinkedIn.

---

## Official documentation

- [smartmontools — USB devices and SAT pass-through](https://www.smartmontools.org/wiki/USB)
- [mke2fs.conf manual — default inode ratio and usage types](https://manpages.ubuntu.com/manpages/noble/man5/mke2fs.conf.5.html)
- [rsync documentation](https://rsync.samba.org/documentation.html)
- [ArchWiki — fstab, external devices and `nofail`](https://wiki.archlinux.org/title/Fstab)

*This is a learning journal based on the steps I actually performed and the tests I completed. Serial numbers, UUIDs, example IP addresses and tailnet names are intentionally omitted.*
