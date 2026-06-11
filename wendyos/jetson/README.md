# Jetson

WendyOS supports the following NVIDIA Jetson targets:

| Board | Machine config | SoC | JetPack / L4T | OTA | Yocto series |
|---|---|---|---|---|---|
| Jetson AGX Orin DevKit (NVMe) | `jetson-agx-orin-devkit-nvme-wendyos` | tegra234 | JetPack 6.2.1 / L4T 36.4.4 | Mender | scarthgap |
| Jetson AGX Orin DevKit (eMMC) | `jetson-agx-orin-devkit-emmc-wendyos` | tegra234 | JetPack 6.2.1 / L4T 36.4.4 | Mender | scarthgap |
| Jetson Orin Nano DevKit (NVMe) | `jetson-orin-nano-devkit-nvme-wendyos` | tegra234 | JetPack 6.2.1 / L4T 36.4.4 | Mender | scarthgap |
| Jetson Orin Nano DevKit (SD) | `jetson-orin-nano-devkit-wendyos` | tegra234 | JetPack 6.2.1 / L4T 36.4.4 | Mender | scarthgap |
| Jetson AGX Thor DevKit (NVMe) | `jetson-agx-thor-devkit-nvme-wendyos` | tegra264 | JetPack 7.1 / L4T 38.4.0 | wendy-update | wrynose |

---

## AGX Thor (tegra264)

The AGX Thor target uses the **wrynose** Yocto series. It is built from a separate layer tree cloned into `repos/wrynose/` alongside the scarthgap tree used by Orin boards.

Key differences from Orin builds:

- **wendy-update OTA** — Thor uses the `wendy-update` A/B OTA client (`WENDYOS_OTA = "wendy"`), not Mender. `meta-mender-tegra` has no wrynose support, so Mender is not used on this board. The OTA stack pulls in `wendyos-update`, `wendyos-data-setup`, and produces a `.wendy` OTA artifact alongside the flashable image.
- **A/B rootfs** — `USE_REDUNDANT_FLASH_LAYOUT_DEFAULT = "1"` enables the upstream meta-tegra A/B rootfs machinery, which inserts the `L4TConfiguration-RootfsRedundancyLevelABEnable.dtbo` overlay and switches to the `_rootfs_ab` flash XML.
- **`/data` partition** — the `data` partition is allocated empty at flash time (no `.dataimg`). `wendyos-data-setup` formats and grows it on first boot.
- **Flash format** — produces `tegraflash-tar` (a compressed tar of the tegraflash package). No Mender artifact; the `.wendy` OTA artifact is produced separately.
- **Bootloader** — uses NVIDIA prebuilt UEFI firmware (`tegra-uefi-prebuilt`).
- **No capsule update** — `WENDYOS_UPDATE_BOOTLOADER = "0"` keeps the Mender capsule artifact off; Thor uses rootfs-only OTA swaps for now.
- **Image name suffix** — wrynose oe-core defaults `IMAGE_NAME_SUFFIX` to `.rootfs`, which would produce deployed symlinks such as `wendyos-image-${MACHINE}.rootfs.tegraflash-tar`. The Thor machine config explicitly sets `IMAGE_NAME_SUFFIX = ""` so the deployed symlink matches the expected name `wendyos-image-${MACHINE}.tegraflash-tar`, consistent with all other boards.
- **Layer set** — Thor's `bblayers.conf` uses `tegra-base.inc` (the shared meta-tegra / meta-tegra-community / meta-tegra-extensions layer set) without the `meta-mender*` layers that Orin builds add on top via `tegra.inc`.

To bootstrap a Thor build:

```bash
./bootstrap.sh --board jetson-agx-thor
```

See `conf/template/boards/jetson-agx-thor/` in the `meta-wendyos` repo for the board template files.
