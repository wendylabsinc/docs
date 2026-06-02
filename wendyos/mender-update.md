# Mender OTA Updates

WendyOS uses [Mender](https://mender.io) for over-the-air (OTA) rootfs A/B updates on supported hardware.

## Supported targets

| Target | Status |
|---|---|
| Jetson AGX Orin (tegra234) | Supported |
| Jetson Orin Nano (tegra234) | Supported |
| Raspberry Pi 3 (64-bit) | Supported (hardware-verified) |
| Raspberry Pi 4 (64-bit) | Supported (hardware-verified) |
| Raspberry Pi 5 (SD and NVMe) | Present in build system; not yet hardware-verified |
| Jetson AGX Thor (tegra264) | Not supported (Phase 1) |
| QEMU | Not supported |

## How it works

Mender maintains two root filesystem partitions (A and B). The active partition is mounted as `/`. On an OTA update:

1. The Mender client downloads the update artifact and writes it to the inactive partition.
2. The bootloader (U-Boot on RPi; UEFI + Mender boot control on Tegra) is updated to boot from the inactive partition on next boot.
3. On successful boot, the new partition is committed as active.
4. On failure, the bootloader rolls back to the previously active partition automatically.

## RPi-specific details

All WendyOS RPi machines use U-Boot as the bootloader. U-Boot 2025.04 (from `meta-lts-mixins`) is required; the `meta-mender-community/meta-mender-raspberrypi` layer provides U-Boot environment integration for A/B slot switching.

The `root=` kernel parameter is managed by U-Boot via the `${mender_kernel_root}` variable, which resolves to the active A/B slot at boot time.

Storage geometry is set per machine:

| Machine | Storage device | Total size |
|---|---|---|
| `raspberrypi3-64-wendyos` | `/dev/mmcblk0` | 7400 MiB |
| `raspberrypi4-64-wendyos` | `/dev/mmcblk0` | 7400 MiB |
| `raspberrypi5-wendyos` | `/dev/mmcblk0` | 7400 MiB |
| `raspberrypi5-nvme-wendyos` | `/dev/nvme0n1` | 32000 MiB |

## Enabling or disabling Mender

`WENDYOS_MENDER` is the master switch. Each RPi machine conf sets it to `"1"`. Override in `local.conf` to change the setting for a specific build:

```bitbake
WENDYOS_MENDER = "0"   # disable Mender for this build
```

For Tegra targets, Mender is on by default for tegra234 and off for tegra264 (Thor Phase 1).

## Build outputs

When `WENDYOS_MENDER = "1"`, the build produces:

- **RPi:** `<image>-<machine>.sdimg` — flash this to the SD card or NVMe drive.
- **Tegra Orin:** `<image>-<machine>.mender` — upload this to the Mender server for OTA delivery; `<image>-<machine>.tegraflash` — used for initial device provisioning.

## Mender client configuration

Key Mender client settings (set in `conf/distro/include/mender.inc`):

| Variable | Value | Description |
|---|---|---|
| `MENDER_SERVER_URL` | configured per deployment | Mender server endpoint |
| `MENDER_TENANT_TOKEN` | configured per deployment | Hosted Mender tenant token |
| `MENDER_UPDATE_POLL_INTERVAL_SECONDS` | `1800` | How often the client polls for updates |
| `MENDER_RETRY_POLL_INTERVAL_SECONDS` | `300` | Retry interval on server error |
| `MENDER_SYSTEMD_AUTO_ENABLE` | `1` | Start Mender client automatically on boot |
| `MENDER_CONNECT_ENABLE` | `1` | Enable Mender Connect (remote terminal) |
