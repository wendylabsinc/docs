# Flashing WendyOS

WendyOS images are flashed to external storage (SD card, NVMe, or eMMC) using `make flash-to-external`.

## Supported targets

| Target | Image format | Device name examples |
|---|---|---|
| Jetson (Orin, Thor) | tegraflash tar | — (uses tegraflash tooling) |
| Raspberry Pi (SD) | Mender `.sdimg` or `.wic` | `mmcblk0`, `sdb` |
| Raspberry Pi 5 (NVMe) | Mender `.sdimg` | `nvme0n1` |

## make flash-to-external

Run from the repo root after a successful build:

```bash
make flash-to-external MACHINE=<machine>
```

### Image selection (RPi)

For Raspberry Pi machines, `flash-to-external` looks for images in this order:

1. `build/tmp/deploy/images/<machine>/<image>-<machine>.sdimg` — Mender A/B image (preferred)
2. `build/tmp/deploy/images/<machine>/<image>-<machine>.rootfs.wic` — plain WIC image (fallback)

If neither is found, the command exits with an error and instructs you to run `make build` first.

For non-RPi machines (Jetson), `flash-to-external` uses the pre-staged image at `deploy/wendyos.img` if present, then falls back to tegraflash tooling.

### Disk selection (Linux)

On Linux, `flash-to-external` lists available block devices including USB, SATA, NVMe, and MMC (`mmcblk`) devices. Enter the device name without the `/dev/` prefix:

```
Enter the disk to flash (e.g., sdb, nvme0n1, mmcblk0) or 'q' to quit:
```

Accepted device name formats: `sdb`, `sdc`, `nvme0n1`, `mmcblk0`, `mmcblk1`, `/dev/sdb`, `/dev/nvme0n1`, `/dev/mmcblk0`.

### Disk selection (macOS)

On macOS, `flash-to-external` lists external physical disks via `diskutil`. Enter the disk identifier (e.g. `disk42`).

## After flashing

After a successful flash, eject the storage and insert it into the target board. No Jetson-specific step is required — the same eject-and-insert flow applies to all supported targets.
