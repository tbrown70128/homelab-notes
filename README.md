# Homelab log: November 25, 2026

Notes from a day of fixing things on the home server. It started with a swap alert at 3 AM and ended with the server on a new SSD. In the afternoon I brought the Raspberry Pi up to date and made it hold a second copy of every backup (sections 6–9).

## Summary

| What | Before | After |
|---|---|---|
| Swap alert | 100% used every night during backups | Alert cleared; `vm.swappiness` lowered to 10 |
| Server boot SSD | Worn-out SK hynix 256 GB (wear level 1/100) | Silicon Power A55 1 TB, healthy |
| Free space on `/` | 112 GB | 788 GB |
| ASUS all-in-one | Stuck on a grey screen with a spinning cursor | Fresh Windows 11 on its Crucial SSD |
| Raspberry Pi | 53 updates pending, package lists 5 weeks old, no automatic updates | Fully updated (kernel 6.18.60, new bootloader), `unattended-upgrades` on, joined the tailnet |
| Copies of the backups | One, on the My Book plugged into the same server | Two: the My Book, plus a nightly copy on the Pi |
| Paperless in backups | Not included | Nightly export, included in Backrest |
| Home Assistant backups | Only inside the VM and HA Cloud (latest backup only) | Also written nightly to the server over NFS, then into Backrest and the Pi |
| Recovery secrets | Restic password only on the server | Restic password, Vaultwarden recovery code and HA encryption key written down off the server |
| Cost | | $0, using drives already on hand |

---

## 1. Swap alert at 3 AM

**Symptom:** Netdata Cloud sent a critical alert: *System swap memory utilization = 100%* (`used_swap`, chart `mem.swap`) at 08:06 UTC.

**Cause:** the nightly Backrest/restic backup runs at 08:00 UTC and reads a lot of files. Linux kept those files in memory as cache, and with the default `vm.swappiness=60` it moved idle program memory (mostly the Home Assistant VM, plus Immich, Jellyfin and Paperless) out to the 4 GB swap file. RAM itself was never short: 8.6 GB was still available and nothing was killed for lack of memory.

**Fix:**
```bash
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swappiness.conf
sudo sysctl -w vm.swappiness=10
```
The swap file lives on the boot SSD, so this also stopped gigabytes of nightly writes to a drive that was already worn out. The alert cleared later that morning.

**Does the server need more RAM?** No. It has 16 GB and uses about 7–8 GB, with the rest as reclaimable cache. Worth revisiting only if "available" memory keeps dropping below about 2 GB or a big new service gets added.

---

## 2. The server's boot SSD was worn out

The server is an ASUS VivoBook X411UN laptop running Ubuntu 24.04. It runs Home Assistant (as a VM), Frigate, Immich, Nextcloud, Jellyfin, Paperless, Vaultwarden, Mealie, Pi-hole, Uptime Kuma, Netdata and others.

SMART on the original SK hynix HFS256G39TND (M.2 2280 SATA):

| Attribute | Value |
|---|---|
| Wear leveling | **1 / 100** (avg ~1,680 erase cycles) |
| Reported uncorrectable errors | 3,126 |
| Offline uncorrectable | 296 |
| Read retries | ~1 billion |
| Total written | 38.7 TB |
| Power-on hours | ~25,300 |

`smartctl -H` still said **PASSED**, which is why you have to read the full attribute table.

### Choosing a replacement

The laptop has two internal drive slots: an **M.2 2280 SATA** slot (boot drive) and a **2.5" bay** (holding a 1 TB Seagate data drive). The replacement had to be **M.2 SATA, not NVMe**.

I compared several 512 GB M.2 SATA drives:

| Drive | Notes | Verdict |
|---|---|---|
| Fikwot FN203 | Lists TLC flash, 300 TBW, LDPC | Best pick to buy |
| Timetec MS06 | Good reviews, ~256 TBW | Backup pick |
| OSCOO ON800 | No flash type or TBW listed | OK |
| KingSpec NT-512 | Pricier, few specs | Skip |
| Silicon Power A55 | NAND varies by batch | Skip if buying new |
| Bestoss S206 | 37 reviews | Skip |
| Transcend MTS830S | Has DRAM, but costs twice as much | Not worth it here |

Lessons:
- Amazon spec tables are often wrong (one listed an M.2 drive as "2.5 inches" and another listed "SMR technology" on an SSD). Trust the title and bullets.
- Don't buy a "256 GB" drive to clone a 256 GB drive: it may be a few bytes smaller, and the clone needs the new drive to be at least as large.
- Moving the OS onto the 2.5" Seagate HDD is possible but a bad idea. It's a 5400 RPM SMR drive, it would be very slow for databases and the VM, and the OS and data would then fail together.

**In the end nothing was bought**, because a spare drive turned up (next section).

---

## 3. The ASUS all-in-one (V241IC) and a drive I'd lost

An old ASUS Vivo AiO V241IC (i5-8250U, 8 GB, touchscreen) had been unplugged for more than two years. It reached the ASUS logo, then showed a grey screen with a spinning cursor.

### Diagnosis
- One RAM slot's latch is broken, but both sticks are detected.
- The BIOS (F2, Aptio Setup) showed **two** SSDs:
  - SATA port 1: Crucial MX500 500 GB (2.5")
  - SATA port 2: **SPCC M.2 SSD 1 TB**. SPCC is Silicon Power. It's hidden under the 2.5" drive box, which is why it wasn't visible at first.
- The boot entry was set to **Disabled**. Setting it to *Windows Boot Manager (SPCC M.2 SSD)* still led to the grey screen, so the hardware was fine and Windows itself was broken.
- Amazon order history (Jan 11, 2023) and a photo from Jan 13, 2023 of Partition Wizard migrating C: confirmed this was a **Silicon Power A55 1 TB** I'd bought and forgotten about.

### Fix
1. Got into Windows Recovery by forcing the power off three times while the grey screen was showing.
2. **Reset this PC → Remove everything → Cloud download → only the Windows drive → just remove files**.
3. Skipped the Microsoft account at setup: `Shift+F10`, then `start ms-cxh:localonly`.
4. Checked the A55 in CrystalDiskInfo: **Good 100%, 372 power-on hours, 0 reallocated, 0 pending, 0 uncorrectable sectors**. It was basically new.
5. Moved Windows onto the Crucial so the A55 could go to the server:
   - Made a Windows 11 USB with the Media Creation Tool and removed the A55.
   - Setup refused the Crucial (*"The selected disk has an MBR partition table"*). Fixed with `Shift+F10`:
     ```
     diskpart
     list disk
     select disk 0      (the 465 GB Crucial, after checking)
     clean
     convert gpt
     exit
     ```
   - Installed Windows 11 on the Crucial.

---

## 4. Desktop drive check (a side discovery)

I ran CrystalDiskInfo on my main desktop while I was at it:

| Drive | Status | Notes |
|---|---|---|
| C: Crucial P3 1 TB NVMe | Good 82% | 51.9 TB written, a lot for a desktop (rated ~220 TBW) |
| D: Crucial MX500 2 TB | Good 99% | Clean |
| E: WD 2 TB USB (5400 RPM) | **Caution** | **1,941 pending sectors**, 46,494 hours |

---

## 5. Cloning the server onto the A55

### Preparation
- Checked that the 08:00 UTC Backrest snapshot existed on the My Book.
- Downloaded `gddrescue` ahead of time to the media drive (`/mnt/media/clone-tools/`), because the live USB has no network once the Anker hub (which carries the server's Ethernet) is unplugged.
- Put the A55 in a **Sabrent** USB enclosure (RTL9210, handles SATA and NVMe). The first USB-A to C cable turned out to be **charge-only**, so the server never saw the drive until I swapped cables.
- Wrote Ubuntu 26.04.1 Desktop to a USB stick with Rufus.

### Gotchas
- The laptop has only **one USB-C port**, and the Anker hub (Ethernet) uses it. Plugging a drive into the hub caused a 3-second network drop. On clone day I unplugged the hub and plugged the enclosure into USB-C (5 Gbps), since the clone doesn't need a network. A USB 2 port would have taken about 2 hours instead of about 20 minutes.
- Switching the server to Wi-Fi wasn't an option: the Home Assistant VM's network is bridged over Ethernet (`br0`), and the Wi-Fi card logs constant PCIe errors.
- The Home Assistant VM ignored ACPI shutdown requests (no guest agent configured). I shut it down from inside HA: Settings → power icon → Shut down system.

### Clone
```bash
# Live USB, after shutting down the HA VM and powering off the server
sudo dpkg -i /media/ubuntu/*/clone-tools/gddrescue_1.27-1_amd64.deb
lsblk -o NAME,SIZE,MODEL,TRAN          # OLD = 238.5G sata SK hynix, NEW = 953.9G usb SPCC
sudo ddrescue -f -n  /dev/OLD /dev/NEW ~/rescue.map
sudo ddrescue -f -r3 /dev/OLD /dev/NEW ~/rescue.map
```
A whole-disk clone copies the partition GUIDs and filesystem UUIDs, so fstab, GRUB and the UEFI boot entry worked without edits. Then I swapped the M.2 drives physically.

### After the swap
```bash
sudo growpart /dev/sdb 3
sudo pvresize /dev/sdb3
sudo lvextend -r -l +100%FREE /dev/ubuntu-vg-1/ubuntu-lv
```
`/` went from 232 GB to **936 GB (788 GB free)**. The boot log showed no disk errors, and SMART on the A55 is clean.

### Things to plug back in after any reboot or hardware change
- **SkyConnect Zigbee/Thread radio** (CP210x, `10c4:ea60`): the HA VM won't start without it (*"Did not find USB device 10c4:ea60"*).
- **WD My Book** (needs its own power adapter): Backrest's restic repository and the homepage container (`dash.home`) both depend on `/mnt/mybook`. Homepage fails to start without it.

---

## 6. Raspberry Pi: updates and Tailscale

The Pi 4 (8 GB, Debian 13 on a 1 TB USB SSD) runs the second Pi-hole and Unbound. Health was fine: 34 days up, 42 °C, never throttled, no failed services. But its package lists hadn't been refreshed since October 21, and **53 updates** were waiting, including security fixes for OpenSSL, Kerberos, NSS, zlib, `sudo` and others, a new kernel (6.18.50 → 6.18.60), and new bootloader firmware.

```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot
sudo rpi-eeprom-update -a     # bootloader: Sep 23 → Oct 30 release
sudo reboot
sudo apt purge linux-image-6.18.50+rpt-rpi-v8 linux-image-6.18.50+rpt-rpi-2712   # old kernel
sudo apt install unattended-upgrades && sudo dpkg-reconfigure -plow unattended-upgrades
```

After the reboot the only errors in the boot log were harmless: `mmc1: Controller never released inhibit bit(s)` (no SD card, since it boots from USB), an ALSA udev rule warning, and one `bluetoothd: Failed to set mode`.

**Tailscale:** installed on the Pi so it doesn't depend on the server's subnet route. Remote SSH still works if the server is down, and the Pi-hole can be used as tailnet DNS.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

**Second Pi-hole confirmed working:** the router hands out the server's Pi-hole first and the Pi's second, the Pi answers queries, and `nebula-sync` on the server pushes a full Teleporter sync from the primary to the Pi every hour.

---

## 7. A second copy of every backup, on the Pi

Backrest's nightly restic backup (08:00 UTC, 7 daily / 4 weekly / 6 monthly) was healthy, but its only copy lived on the My Book, **which is plugged into the same server**. Theft, a power surge or ransomware on the server could take out the data and the backups together.

Fix: the Pi pulls a mirror of the whole restic repository every morning at 05:30 Eastern (10:30 UTC, well after the backup finishes).

- A dedicated SSH key on the Pi (`pi-backup`), added to the server's `authorized_keys`. It connects over Tailscale.
- `~/bin/pull-laptop-backup.sh` on the Pi, run from the Pi's crontab:
  ```bash
  rsync -a --delete --partial --exclude='locks/' \
        --backup --backup-dir="$OLD/$STAMP" \
        server:/mnt/mybook/restic-repo/ ~/backups/server-restic/
  ```
  Anything the server deletes or changes (for example after a `prune`) is kept for 30 days in `~/backups/server-restic-old/<date>`, so a damaged or wiped repo on the server doesn't instantly wipe the Pi's copy too.
- Logs go to `~/backups/logs/`, and `~/backups/LAST_SUCCESS` holds the time of the last good run.
- The repo is already encrypted by restic, so the Pi only ever holds ciphertext.

**First run:** 42 GB in about 10 minutes. The copy matches the source exactly (2,516 files, 44,060,860,636 bytes, 8 snapshots), and a dry-run `rsync` afterwards found nothing left to transfer.

Gotchas:
- The first attempt to add the key ran on the Pi instead of the server. Check the prompt (`@server` vs `@pi`) before pasting.
- The server runs on UTC and the Pi on Eastern time, so check the timezone before scheduling one job after another.

---

## 8. Paperless and Home Assistant added to the backups

**Paperless** wasn't in the Backrest plan at all. I followed the same pattern as Immich and Nextcloud: back up the files, leave out the live database folder, and rely on a proper export instead.

- Server crontab, 07:30 UTC (half an hour before Backrest):
  ```
  30 7 * * * docker exec paperless-webserver-1 document_exporter ../export --no-progress-bar >> $HOME/paperless/export.log 2>&1
  ```
- Backrest plan `server-data`: added `/mnt/media/paperless`, excluded `/mnt/media/paperless/pgdata` and `/mnt/media/paperless/redisdata`. Saved `config.json.bak-2026-11-25` first, then restarted the container. No errors, and the next run was scheduled normally.
- The exporter warns that sensitive fields are in plaintext without a passphrase. That's acceptable here, because the export only leaves the server inside the encrypted restic repo.

**Home Assistant** backups only existed inside the VM and in HA Cloud, which keeps just the latest one. A dead server would have taken the rest.

- On the server: installed `nfs-kernel-server` and exported `/mnt/media/backups/homeassistant` to the HA VM's address only (`rw,sync,all_squash,anonuid=1000`). Opened UFW port 2049/tcp from that one address. Backrest already covers `/mnt/media/backups`, so nothing else needed changing.
- Reserved the HA VM's address in the router, so the share can't break from a DHCP change.
- In HA: Settings → System → Storage → Add network storage (`laptop_backups`, Backup, NFS), then enabled it as a location under Backup settings.
- First backup: 742 MB, a valid tar, readable by the Backrest container.

---

## 9. Vaultwarden two-step lockout

When I went to save the restic password in Vaultwarden, it asked for an authenticator code, and no authenticator app had a "Vaultwarden" entry. There's no admin panel (no `ADMIN_TOKEN`), so the options were the recovery code or editing the database by hand.

- The **recovery code** worked. Using it turns two-step login **off**, so I turned it back on straight away (Settings → Security → Two-step login → Authenticator app) and saved the new recovery code.
- Lesson: Bitwarden-compatible servers usually show up in authenticator apps as **"Bitwarden"**, not "Vaultwarden".

Now written down off the server: the **restic/Backrest repository password**, the **Vaultwarden recovery code** and the **Home Assistant backup encryption key**. With those three, everything can be restored from the Pi's copy even if the server is gone.

---

## Open items

- [ ] Keep the old SK hynix SSD untouched for a few weeks as a fallback, then retire it. Don't trust it with data.
- [x] Store the restic repository password somewhere **off** the server. *Done: written down offline, along with the Vaultwarden recovery code and the HA encryption key.*
- [ ] Check that the first full overnight run worked: Paperless and HA in the 08:00 UTC snapshot, and `~/backups/LAST_SUCCESS` on the Pi updated after 05:30 Eastern.
- [ ] Do a real restore test from the Pi's copy (for example, restore Vaultwarden into a scratch container).
- [ ] Get a copy out of the house. The Pi is a separate device but sits in the same home.
- [ ] Find out what's writing so much to the desktop's Crucial P3.
- [ ] Decide what the ASUS all-in-one is for (kitchen screen for Mealie and Home Assistant? Jellyfin player?).
- [ ] Optionally add the movie and TV folders on `/mnt/media` to the backups.
