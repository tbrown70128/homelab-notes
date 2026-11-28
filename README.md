# Homelab log

Maintenance notes for my home server (an old ASUS laptop running Ubuntu, Docker and a Home Assistant VM) and a Raspberry Pi 4 that runs the second Pi-hole and keeps a copy of the backups. Newest first.

| Date | What happened |
|---|---|
| [2026-11-28](logs/2026-11-28.md) | Server overheated and powered off (stuck Nextcloud job); temperature alerts; automatic power-on after a sudden power-off; hardware watchdog; down and internet-outage alerts from the Pi and healthchecks.io |
| [2026-11-26](logs/2026-11-26.md) | Overnight crash; Tailscale failover through the Pi; automatic power-on after power cuts (Wake-on-LAN didn't work, the clock alarm did); network watchdog |
| [2026-11-25](logs/2026-11-25.md) | Swap alert; worn-out boot SSD replaced; all-in-one repair; Pi updates; a second copy of every backup |

## Scripts

| Script | What it does |
|---|---|
| [`scripts/power-guard`](scripts/power-guard) | Shuts the laptop down at low battery with a clock alarm set, and keeps it off until wall power returns |
| [`scripts/netwatch`](scripts/netwatch) | Resets the network adapter, then reboots, when the router is unreachable; never reboots for an ISP outage |
| [`scripts/heat-alert`](scripts/heat-alert) | Discord alert when the CPU runs hot, naming the top CPU users; clean shutdown at 98°C |
| [`scripts/wake-arm`](scripts/wake-arm) | Rolling clock alarm so the server powers itself back on ~10 min after a sudden power-off; deliberate shutdowns stay off; stops after 3 in 6 hours |
| [`scripts/server-watch`](scripts/server-watch) | Runs on the Pi: Discord alerts when the server goes down or comes back, and reports internet outages (with any missed alerts) once the connection returns |

The systemd units are in [`scripts/systemd/`](scripts/systemd/) (the Pi's user units in [`scripts/systemd/user/`](scripts/systemd/user/)). Config files with webhooks and ping URLs are kept off GitHub; the `*.conf.example` files show their format.
