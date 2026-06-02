# WendyOS on Raspberry Pi

WendyOS supports **Raspberry Pi 5**, **Raspberry Pi 4**, and **Raspberry Pi 3** with SD card boot configurations. Raspberry Pi 5 also supports NVMe boot configurations.

## Mender OTA

Mender OTA is enabled on RPi3 and RPi4 (hardware-verified). RPi5 Mender support is present in the build system but has not yet been hardware-verified and may require adjustments.

All Mender-enabled RPi machines use U-Boot and produce a Mender `.sdimg` (A/B rootfs layout managed by `mender-image-sd`). U-Boot 2025.04 is required and is supplied by the `meta-lts-mixins` layer (backported into the scarthgap build).

Each machine conf sets `WENDYOS_MENDER = "1"` explicitly. The distro-level default remains `"0"` for the `rpi` MACHINEOVERRIDES group.

## Partition Layout

Mender-enabled RPi images use a 4-partition A/B layout managed by `mender-image-sd`:

| Partition | Role | Size | FS |
|---|---|---|---|
| 1 | Boot (U-Boot + firmware) | 100 MB | FAT32 |
| 2 | Root A (active) | derived from total size | ext4 |
| 3 | Root B (inactive) | derived from total size | ext4 |
| 4 | Data (`/data`) | 256 MB | ext4 |

Storage geometry defaults (SD card, `/dev/mmcblk0`):

| Variable | Default |
|---|---|
| `MENDER_STORAGE_DEVICE` | `/dev/mmcblk0` |
| `MENDER_BOOT_PART_SIZE_MB` | `100` |
| `MENDER_DATA_PART_SIZE_MB` | `256` |
| `MENDER_STORAGE_TOTAL_SIZE_MB` | `7400` (fits a nominal 8 GB SD card) |

The `raspberrypi5-nvme-wendyos` machine overrides `MENDER_STORAGE_DEVICE` to `/dev/nvme0n1` and `MENDER_STORAGE_TOTAL_SIZE_MB` to `32000`.

The build output is a `.sdimg` file located at:

```
build/tmp/deploy/images/<machine>/<image>-<machine>.sdimg
```

`make flash-to-external` automatically uses the `.sdimg` when present, falling back to a `.wic` if no `.sdimg` is found.

## Config partition

See [Config Partition](../config-partition.md) for full details and usage guidance.

## Build configuration

| Variable | Default | Description |
|---|---|---|
| `WENDYOS_CONFIG_PART_SIZE_MB` | `64` | Size of the FAT32 config partition in MB |
| `WENDYOS_RPI_UBOOT` | `1` | Use U-Boot (required for Mender A/B updates) |
| `MENDER_STORAGE_TOTAL_SIZE_MB` | `7400` | Total Mender storage budget in MiB |

Set these in your `local.conf` or machine conf to override the defaults.
