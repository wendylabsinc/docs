# WendyOS on Raspberry Pi

WendyOS supports **Raspberry Pi 5** with SD card and NVMe boot configurations.

## Partition Layout

RPi5 images use WIC (OpenEmbedded Image Creator) with a GPT partition table.

### SD card layout

| Partition | Label | Size | FS | Mount |
|---|---|---|---|---|
| boot | boot | 128 MB | FAT32 | `/boot` |
| config | config | 64 MB* | FAT32 | `/config` |
| root | root | 8 GB | ext4 | `/` |

### NVMe layout

Same as SD card, but written to `nvme0n1` instead of `mmcblk0`.

\* Default size. See [Build configuration](#build-configuration).

## Config partition

See [Config Partition](../config-partition.md) for full details and usage guidance.

## Build configuration

| Variable | Default | Description |
|---|---|---|
| `WENDYOS_CONFIG_PART_SIZE_MB` | `64` | Size of the FAT32 config partition in MB |

Set this in your `local.conf` or distro config to override the default.
