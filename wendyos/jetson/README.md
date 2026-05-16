# Jetson

WendyOS supports the following NVIDIA Jetson targets:

| Board | Machine config | SoC | JetPack / L4T | Mender OTA | Yocto series |
|---|---|---|---|---|---|
| Jetson AGX Orin DevKit (NVMe) | `jetson-agx-orin-devkit-nvme-wendyos` | tegra234 | JetPack 6.2.1 / L4T 36.4.4 | Yes | scarthgap |
| Jetson AGX Orin DevKit (eMMC) | `jetson-agx-orin-devkit-emmc-wendyos` | tegra234 | JetPack 6.2.1 / L4T 36.4.4 | Yes | scarthgap |
| Jetson Orin Nano DevKit (NVMe) | `jetson-orin-nano-devkit-nvme-wendyos` | tegra234 | JetPack 6.2.1 / L4T 36.4.4 | Yes | scarthgap |
| Jetson Orin Nano DevKit (SD) | `jetson-orin-nano-devkit-wendyos` | tegra234 | JetPack 6.2.1 / L4T 36.4.4 | Yes | scarthgap |
| Jetson AGX Thor DevKit (NVMe) | `jetson-agx-thor-devkit-nvme-wendyos` | tegra264 | JetPack 7.1 / L4T 38.4.0 | No (Phase 1) | wrynose |

---

## AGX Thor (tegra264)

The AGX Thor target uses the **wrynose** Yocto series. It is built from a separate layer tree cloned into `repos/wrynose/` alongside the scarthgap tree used by Orin boards.

Key differences from Orin builds:

- **No Mender OTA** — Mender on Thor is deferred. `WENDYOS_MENDER = "0"` is set automatically for tegra264. There is no A/B partition layout and no `/data` partition in Phase 1.
- **Flash format** — produces `tegraflash-tar` (a compressed tar of the tegraflash package) rather than separate `tegraflash` + `mender` artefacts.
- **Bootloader** — uses NVIDIA prebuilt UEFI firmware (`tegra-uefi-prebuilt`).
- **Image name suffix** — wrynose oe-core defaults `IMAGE_NAME_SUFFIX` to `.rootfs`, which would produce deployed symlinks such as `wendyos-image-${MACHINE}.rootfs.tegraflash-tar`. The Thor machine config explicitly sets `IMAGE_NAME_SUFFIX = ""` so the deployed symlink matches the expected name `wendyos-image-${MACHINE}.tegraflash-tar`, consistent with all other boards.

To bootstrap a Thor build:

```bash
./bootstrap.sh --board jetson-agx-thor
```

See `conf/template/boards/jetson-agx-thor/` in the `meta-wendyos` repo for the board template files.
