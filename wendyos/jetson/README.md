# WendyOS on Jetson

WendyOS supports NVIDIA Jetson devices, currently targeting the **Jetson Orin Nano Devkit** with NVMe and SD card boot configurations.

## Partition Layout

Jetson uses NVIDIA's Tegra partition XML format. WendyOS modifies the BSP-provided layouts to add Mender A/B update support and the `config` partition.

### NVMe layout

| Partition | ID | Size | FS | Notes |
|---|---|---|---|---|
| *(QSPI boot partitions)* | — | — | — | NVIDIA BSP: UEFI, DTB, etc. |
| UDA | — | — | — | NVIDIA compatibility, unused by WendyOS |
| APP | p1 | 8 GB | ext4 | Root filesystem A |
| APP_b | p2 | 8 GB | ext4 | Root filesystem B (Mender A/B) |
| config | p16 | 64 MB* | FAT32 | Configuration partition |
| mender_data | p17 | 512 MB* | ext4 | Persistent data, auto-expands on first boot |

### SD card layout

The SD card layout mirrors the NVMe layout above, using `mmcblk0` partitions instead of NVMe block devices.

\* Default sizes. See [Build configuration](#build-configuration).

## Config partition

The `config` partition (FAT32, label `config`) is mounted at `/config` on boot via an fstab entry with `nofail`. It is readable and writable on Linux, macOS, and Windows without additional drivers.

Use it to provide configuration files that need to survive OS updates or that users should be able to edit from a host machine without booting the device.

## Mender data partition

The `mender_data` partition (ext4, partition p17) holds Mender state and persistent application data. It is positioned after `APP_b` so it can be expanded to fill remaining disk space.

On first boot, `mender-grow-data.service` automatically expands the partition to use all available free space after the partition table is written.

> **Note:** WendyOS uses `mender_data` (p17) for persistent storage, not `UDA` (p15). UDA is kept in the partition table for NVIDIA compatibility but is not mounted.

## Build configuration

| Variable | Default | Description |
|---|---|---|
| `WENDYOS_CONFIG_PART_SIZE_MB` | `64` | Size of the FAT32 config partition in MB |
| `WENDYOS_CONFIG_PART_NUMBER` | `16` | GPT partition number for config |
| `MENDER_DATA_PART_NUMBER` | `17` | GPT partition number for mender\_data |

Set these in your `local.conf` or distro config to override defaults.
