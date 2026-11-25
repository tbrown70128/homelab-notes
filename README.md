# Homelab log: November 25, 2026

Notes from a day of fixing things on the home server. It started with a swap alert at 3 AM and ended with the server on a new SSD.

## Summary

| What | Before | After |
|---|---|---|
| Swap alert | 100% used every night during backups | Alert cleared; `vm.swappiness` lowered to 10 |
| Server boot SSD | Worn-out SK hynix 256 GB (wear level 1/100) | Silicon Power A55 1 TB, healthy |
| Free space on `/` | 112 GB | 788 GB |
| ASUS all-in-one | Stuck on a grey screen with a spinning cursor | Fresh Windows 11 on its Crucial SSD |
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

**To do:** copy any irreplaceable videos off E:. It's failing. Use robocopy so it skips unreadable files and logs them:
```
robocopy E:\ <destination> /E /R:1 /W:1 /XD "System Volume Information" "$RECYCLE.BIN" /LOG:<destination>\WD-copy-log.txt /TEE
```

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

## Open items

- [ ] Keep the old SK hynix SSD untouched for a few weeks as a fallback, then retire it. Don't trust it with data.
- [ ] Store the restic repository password somewhere **off** the server. A copy in Vaultwarden alone isn't enough, since Vaultwarden runs on the same server.
- [ ] Find out what's writing so much to the desktop's Crucial P3.
- [ ] Decide what the ASUS all-in-one is for (kitchen screen for Mealie and Home Assistant? Jellyfin player?).
- [ ] Optionally add the movie and TV folders on `/mnt/media` to the backups.
