# Config Partition

Every WendyOS image includes a small FAT32 partition labelled `config`, mounted at `/config` on the device. It is readable and writable on Linux, macOS, and Windows without any additional drivers or tools — including immediately after `dd`-ing an image onto an SD card or NVMe drive.

## Why it exists

The root filesystem on a WendyOS device is replaced wholesale during OTA updates. Anything written to `/` will be overwritten when an update is applied.

The `config` partition is outside the update boundary. Files placed there survive OS updates and are available from the very first boot. This makes it the right place for anything that needs to be pre-staged before a device has ever booted, or preserved across updates.

## Accessing it from a host computer

Because the partition is FAT32, your computer mounts it automatically when you plug in the storage.

- **macOS / Linux:** appears as a volume labelled `config`
- **Windows:** appears as a drive labelled `config`

Write files there, eject, and they are available at `/config` on the device on next boot.

## Accessing it on the device

The partition is mounted at `/config` on every boot (fstab `nofail`, so a missing or unformatted partition is not fatal):

```sh
ls /config
```

## wendy.conf

The primary intended use of the config partition is a `wendy.conf` file that WendyOS reads on boot to configure the device — setting the device name, enrolling with the cloud, configuring WiFi, and so on. `wendy os install` will write this file automatically after flashing.

> **Status:** The `wendy.conf` systemd dispatcher and handler infrastructure is under active development (WDY-779). Once shipped, this will be the recommended way to provision a device without ever needing to SSH in.

## Size

The partition is **64 MB** by default. This is intentionally small — it is for configuration, not application data or logs. Large or frequently-written data belongs on the data partition (`/data`).

The size can be adjusted at build time via `WENDYOS_CONFIG_PART_SIZE_MB` in your distro or `local.conf`.
