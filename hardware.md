# Hardware

## Specification

| Component | Detail |
|---|---|
| Board | Raspberry Pi 3 Model B Plus Rev 1.3 |
| CPU | 4x ARM Cortex-A53 (aarch64), 600 MHz to 1.4 GHz |
| Memory | 1 GB (904 MiB usable) |
| Boot and root storage | 59.7 GB microSD |
| Wired network | `eth0`, 100 Mb/s full duplex |
| Peripherals | HP LaserJet 1020 on USB, connected on demand |

## Storage layout

| Device | Size | Filesystem | Mount point |
|---|---|---|---|
| `mmcblk0p1` | 512 MB | vfat | `/boot/firmware` |
| `mmcblk0p2` | 59.2 GB | ext4 | `/` |

Swap: 904 MB zram plus a 904 MB file-backed swap.

## Resource snapshot

Point-in-time reading with Pi-hole, three Docker containers, Samba, Cockpit and Tailscale running. Not a benchmark.

| Metric | Value |
|---|---|
| Memory used | 504 MiB of 904 MiB |
| Swap used | 138 MiB |
| Root filesystem | 7.2 GB used of 59 GB (13%) |
| Temperature | 38.1 °C |

## Planned: external storage

An external drive will provide storage for Samba, media and downloads. This is postponed until a better power supply is bought. The sequence:

1. Identify the device with `lsblk` and `dmesg`, and confirm its capacity.
2. Inspect existing partitions and filesystems before changing anything.
3. Mount persistently through `/etc/fstab` using the filesystem UUID.
4. Share the mount through Samba.
