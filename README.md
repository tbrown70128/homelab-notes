# Homelab log

Maintenance notes for my home server (an old ASUS laptop running Ubuntu, Docker and a Home Assistant VM) and a Raspberry Pi 4 that runs the second Pi-hole and keeps a copy of the backups. Newest first.

| Date | What happened |
|---|---|
| [2026-11-26](logs/2026-11-26.md) | Overnight crash; Tailscale failover through the Pi; automatic power-on after power cuts (Wake-on-LAN didn't work, the clock alarm did); network watchdog |
| [2026-11-25](logs/2026-11-25.md) | Swap alert; worn-out boot SSD replaced; all-in-one repair; Pi updates; a second copy of every backup |

## Scripts

| Script | What it does |
|---|---|
| [`scripts/power-guard`](scripts/power-guard) | Shuts the laptop down at low battery with a clock alarm set, and keeps it off until wall power returns |
| [`scripts/netwatch`](scripts/netwatch) | Resets the network adapter, then reboots, when the router is unreachable; never reboots for an ISP outage |

The systemd units are in [`scripts/systemd/`](scripts/systemd/).
