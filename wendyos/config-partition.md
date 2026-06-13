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

The partition is mounted at `/config` on every boot (fstab `nofail`, so a missing or unformatted partition is not fatal). After the first-boot reclaim (see below), the partition no longer exists and the mount silently does nothing.

```sh
ls /config
```

## First-boot reclaim (Raspberry Pi)

On Raspberry Pi, the `config` partition is provisioned by Mender as an extra partition placed after `/data`. Because it sits after `/data`, `mender-growfs-data` cannot grow `/data` to fill the card. Instead, `reclaim-config-part` runs as a one-shot systemd service on first boot:

1. Waits for `wendy-agent` to consume and erase the seed files from `/config`.
2. Deletes the `config` partition from the partition table.
3. Grows `/data` (and its ext4 filesystem) to fill the rest of the storage device.

The service is resumable: if power is lost between the partition delete and the resize, the next boot completes the grow. A done-stamp at `/data/.wendyos-reclaim-config.done` prevents the service from running again once `/data` fills the device.

After reclaim, the `/config` mount point remains in fstab with `nofail`, so the absence of the partition does not block boot.

## wendy-agent self-update

If a file named `wendy-agent` is present in `/config` on boot, the agent validates it (must be a 64-bit ELF binary for the device's architecture) and, if valid, installs it to `/usr/local/bin/wendy-agent` and exits so systemd restarts it with the new binary. The file is deleted from `/config` regardless of outcome, so it is only applied once.

`wendy os install` writes the latest stable arm64 agent binary to the config partition automatically after flashing.

## wendy.conf

`wendy.conf` is an INI-format file the agent reads on first boot to configure the device. If present, the agent applies its contents and then **deletes the file** so settings are not re-applied on subsequent boots.

### Format

```ini
[wifi]
ssid = MyNetwork
password = hunter2
```

The `[wifi]` section connects the device to a WiFi network. `password` may be omitted for open networks.

### How it is applied

On every boot, `wendy-agent` checks for `/config/wendy.conf`. If found:

1. If `[wifi]` contains a non-empty `ssid`, runs `nmcli device wifi connect <ssid> [password <password>]` to connect and create a persistent NetworkManager profile.
2. Deletes `/config/wendy.conf` — even on failure, so bad credentials are not retried on every boot.

`wendy os install` writes this file automatically when you supply WiFi credentials (via `--wifi-ssid` / `--wifi-password` flags or the interactive prompt).

## Size

The partition is **128 MB**. On Raspberry Pi this space is reclaimed into `/data` after the first boot (see [First-boot reclaim](#first-boot-reclaim-raspberry-pi) above).
