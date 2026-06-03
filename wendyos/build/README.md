# WendyOS Build System

WendyOS is built with [Yocto Project](https://www.yoctoproject.org/) / OpenEmbedded (OE), releasing on the **Scarthgap** branch. Builds target ARM64 hardware and run inside a Docker container (Ubuntu 24.04 LTS) so macOS and Linux hosts produce identical output.

## Supported Targets

| Board ID (`BOARD=`) | Yocto `MACHINE` | Hardware | Boot |
|---|---|---|---|
| `rpi5-sd` | `raspberrypi5-wendyos` | Raspberry Pi 5 | SD card |
| `rpi5-nvme` | `raspberrypi5-nvme-wendyos` | Raspberry Pi 5 | NVMe |
| `jetson-orin-nano-nvme` | `jetson-orin-nano-devkit-nvme-wendyos` | Jetson Orin Nano | NVMe |
| `jetson-orin-nano-sd` | `jetson-orin-nano-devkit-wendyos` | Jetson Orin Nano | SD |
| `jetson-agx-orin` | `jetson-agx-orin-devkit-nvme-wendyos` | Jetson AGX Orin | NVMe |
| `jetson-agx-orin-emmc` | `jetson-agx-orin-devkit-emmc-wendyos` | Jetson AGX Orin | eMMC |
| `qemu-arm64` | `qemuarm64-wendyos` | QEMU ARM64 | — |

The meta-layers for the legacy-style RPi-only and VM-only repos (`meta-wendyos-rpi`, `meta-wendyos-virtual`) use their own distros (`edgeos-rpi`, `edgeos-vm`) and machine names (`raspberrypi4-64-edgeos`, `raspberrypi5-edgeos`, `wendyos-vm-arm64`). The unified `wendyos` repo (described in this guide) is the canonical build path.

---

## Host Requirements

### Common (Linux and macOS)

- Git
- Docker (running)
- ~100 GB free disk space (150 GB recommended for multiple targets)
- Reliable internet connection (source tarballs are fetched at build time)

### Linux

The build user must be in the `docker` group:

```bash
sudo usermod -aG docker $USER
# log out and back in
```

Ubuntu 24.04 restricts unprivileged user namespaces, which BitBake needs. The Docker image already handles this for container builds, but for host builds the install script applies the needed sysctl override automatically.

### macOS (Docker Desktop)

- Docker Desktop 4.0 or later
- Allocate sufficient resources in Docker Desktop → Settings → Resources:
  - Memory: 8 GB minimum (16 GB+ recommended)
  - CPUs: 4+ cores
  - Disk: 150 GB minimum
- Enable VirtioFS in Docker Desktop settings for better build performance

On macOS, build artifacts live in Docker volumes (not on the host filesystem) to work around the case-insensitive HFS+/APFS default. `make deploy` copies the finished flash package back to the host after a successful build.

---

## Installing kas

`kas` is a Yocto configuration helper maintained by Siemens. It is **not required** for the `wendyos` repo — that repo uses `bootstrap.sh` + `make`. The `meta-wendyos-rpi` and `meta-wendyos-virtual` repos also use `bootstrap.sh` directly.

If you want to use `kas` independently:

```bash
pip install kas
# or
pipx install kas
```

---

## First Build (wendyos repo)

### 1. Clone the repository

```bash
mkdir -p ~/build/wendyos
cd ~/build/wendyos
git clone git@github.com:wendylabsinc/meta-wendyos.git meta-wendyos
cd meta-wendyos
```

### 2. Bootstrap for your target board

```bash
BOARD=rpi5-sd ./bootstrap.sh
```

Bootstrap performs these steps:
- Validates that `meta-wendyos` is inside the working directory (required for the Docker volume mount)
- Clones all upstream Yocto layers into `repos/<tree>/` at the pinned SRCREVs, where `<tree>` is the value of `WENDYOS_LAYER_TREE` (default `scarthgap`)
- Copies `conf/template/boards/<board-id>/local.conf` and `bblayers.conf` into `build/conf/`
- Writes `build/.wendyos-env` exporting `WENDYOS_LAYER_TREE` for the active board
- Creates the Docker build image (`wendyos-build:latest`)

The Yocto core (`bitbake`, `openembedded-core`, `meta-yocto`) is composed from upstream split repos rather than a bundled `poky.git` monolith. Inside each tree the repos are flat siblings — there is no `poky/` umbrella directory.

Available board IDs are the directory names under `conf/template/boards/`.

### 3. (Optional) Customise `build/conf/local.conf`

Key variables you may want to change before building — see [configuration.md](configuration.md) for the full reference.

### 4. Build

```bash
make build
```

On macOS the build runs inside the Docker container automatically. On Linux you can also run it directly:

```bash
. ./build/.wendyos-env
. ./repos/$WENDYOS_LAYER_TREE/openembedded-core/oe-init-build-env build
bitbake wendyos-image
```

First build: roughly 1–4 hours depending on host speed and network. Subsequent builds using a warm `sstate-cache` typically complete in 5–15 minutes.

### 5. Find the output

```
build/tmp/deploy/images/<machine>/
```

| File | Description |
|---|---|
| `wendyos-image-<machine>.sdimg` | Mender A/B SD image (RPi with Mender enabled) |
| `wendyos-image-<machine>.rootfs.wic` | Raw WIC disk image (RPi without Mender / QEMU) |
| `wendyos-image-<machine>.rootfs.wic.bmap` | Block map for fast flashing |
| `wendyos-image-<machine>.tegraflash.tar.gz` | Complete Tegra flash package |
| `wendyos-image-<machine>.mender` | Mender OTA artifact |

On macOS, artifacts live inside Docker volumes. Use `make deploy` to copy the Tegra flash package to the host `deploy/` directory.

---

## Make Targets (wendyos repo)

| Target | Description |
|---|---|
| `make setup BOARD=<id>` | First-time setup: clone repos, create Docker image |
| `make build` | Build `wendyos-image` for the current `MACHINE` |
| `make build MACHINE=<yocto-name>` | Build for a specific Yocto machine |
| `make build-sdk` | Build the SDK for application cross-compilation |
| `make shell` | Open an interactive shell inside the build container |
| `make deploy` | Copy Tegra flash tarball from Docker volume to `./deploy/` (macOS) |
| `make flash-to-external` | Interactively create a flashable image and write to a drive |
| `make clean` | Remove `build/tmp/` and `build/cache/` (keeps downloads and sstate) |
| `make distclean` | Remove everything including downloads (full cold rebuild) |
| `make volumes-create` | Create Docker volumes for case-sensitive storage (macOS) |
| `make volumes-remove` | Delete Docker volumes (destroys all cached build data) |

---

## First Build (meta-wendyos-rpi)

```bash
cd meta-wendyos-rpi
./bootstrap.sh

. ./build/.wendyos-env
. ./repos/$WENDYOS_LAYER_TREE/openembedded-core/oe-init-build-env build
bitbake edgeos-rpi-image
```

Output in `build/tmp/deploy/images/<machine>/`:
- `.wic.bz2` — compressed SD card image
- `.wic.bmap` — block map for bmaptool flashing
- `.ext4` — raw root filesystem

Flash to SD card:

```bash
sudo bmaptool copy edgeos-rpi-image-*.wic.bz2 /dev/sdX
```

---

## First Build (meta-wendyos-virtual)

```bash
cd meta-wendyos-virtual
./bootstrap.sh

. ./build/.wendyos-env
. ./repos/$WENDYOS_LAYER_TREE/openembedded-core/oe-init-build-env build
bitbake edgeos-vm-image
```

Output in `build/tmp/deploy/images/wendyos-vm-arm64/`:
- `.wic.qcow2` — for UTM / Lima / QEMU on macOS
- `.wic.vmdk` — for VMware Fusion
- `.ext4` — raw filesystem image

---

## Speeding Up Rebuilds

Set a persistent `DL_DIR` path in `build/conf/local.conf`:

```bitbake
DL_DIR = "/shared/yocto/downloads"
```

`SSTATE_DIR` is automatically partitioned per layer tree (`sstate-cache/${WENDYOS_LAYER_TREE}`) so builds targeting different Yocto series on the same workspace do not share sstate. Source tarballs (`DL_DIR`) are shared across series — they are keyed by URL and checksum and are always reusable. The CI pipeline uses S3-backed sstate with an EBS snapshot L1 cache; see `docs/build-cache.md` in the `wendyos` repo for the setup guide.
