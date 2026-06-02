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

## Ribbon cameras (CSI)

WendyOS applies NVIDIA camera Device Tree overlays at boot for Orin boards so that the kernel IMX drivers probe correctly. The overlays are supplied by `nvidia-kernel-oot` and installed into `/boot/` by `l4t-launcher-extlinux`. L4TLauncher reads the resulting `OVERLAYS` line in `extlinux.conf` and applies the overlays during boot.

### Default overlay assignments

| Board(s) | Default overlay | Sensors |
|---|---|---|
| Jetson Orin Nano DevKit (SD and NVMe) | `tegra234-p3767-camera-p3768-imx219-imx477.dtbo` | IMX219 in CAM0, IMX477 in CAM1 |
| Jetson AGX Orin DevKit (eMMC and NVMe) | `tegra234-p3737-camera-dual-imx274-overlay.dtbo` | Dual IMX274 |

The Orin Nano default assumes the Raspberry Pi-ecosystem ribbon cameras (IMX219 + IMX477) on the P3768 dual-CSI carrier. If your physical wiring has the sensors swapped, override in `local.conf`:

```bitbake
UBOOT_EXTLINUX_FDTOVERLAYS:jetson-orin-nano-devkit = "tegra234-p3767-camera-p3768-imx477-A.dtbo"
```

The AGX Orin default targets the NVIDIA-reference dual-IMX274 module. If no camera is attached the overlay fails to probe without causing a boot regression. Override in `local.conf` for other sensor modules.

Camera overlays are not set for AGX Thor (tegra264 / JP7) — that target is deferred.

### Verifying camera bring-up

After flashing, confirm the overlays are active:

```sh
# DT nodes should appear
find /proc/device-tree -iname 'imx*'

# Driver probe messages
dmesg | grep -E 'imx477|imx219|nv_imx|tegra-capture-vi'

# Video and sub-device nodes
ls /dev/video* /dev/v4l-subdev*

# WendyOS camera list
wendy device camera list
```

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
