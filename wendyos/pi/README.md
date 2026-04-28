# WendyOS on Raspberry Pi

WendyOS supports **Raspberry Pi 5**, **Raspberry Pi 4**, and **Raspberry Pi 3** with SD card boot configurations. Raspberry Pi 5 also supports NVMe boot configurations.

## Partition Layout

Raspberry Pi images use WIC (OpenEmbedded Image Creator) with a GPT partition table.

### SD card layout

| Partition | Label | Size | FS | Mount |
|---|---|---|---|---|
| boot | boot | 128 MB | FAT32 | `/boot` |
| config | config | 64 MB* | FAT32 | `/config` |
| root | root | 8 GB | ext4 | `/` |

### NVMe layout

Raspberry Pi 5 supports NVMe images with the same partition layout, written to `nvme0n1` instead of `mmcblk0`.

\* Default size. See [Build configuration](#build-configuration).

## Config partition

See [Config Partition](../config-partition.md) for full details and usage guidance.

## Build configuration

| Variable | Default | Description |
|---|---|---|
| `WENDYOS_CONFIG_PART_SIZE_MB` | `64` | Size of the FAT32 config partition in MB |

Set this in your `local.conf` or distro config to override the default.
