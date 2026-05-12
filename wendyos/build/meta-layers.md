# Meta-Layers

WendyOS is built from a set of stacked Yocto / OpenEmbedded meta-layers. Every build begins with Poky (the reference OE distribution) and adds hardware BSP layers, then the WendyOS application layer on top.

Layers declare compatibility with `LAYERSERIES_COMPAT_wendyos = "scarthgap wrynose"`. Orin, RPi, and QEMU boards build on **scarthgap**; AGX Thor builds on **wrynose**.

---

## Layers by Target

### wendyos (unified repo)

Layers are assembled per board by `bblayers.conf` fragments under `conf/template/include/bblayers/`. The full set for a Jetson + RPi + QEMU build is:

| Layer | Source | Collection Name | Purpose |
|---|---|---|---|
| `poky/meta` | `git.yoctoproject.org/poky` | `core` | OpenEmbedded core |
| `poky/meta-poky` | `git.yoctoproject.org/poky` | `yocto` | Poky distro policy |
| `poky/meta-yocto-bsp` | `git.yoctoproject.org/poky` | `yoctobsp` | Reference BSPs |
| `meta-openembedded/meta-oe` | `github.com/openembedded/meta-openembedded` | `openembedded-layer` | Additional recipes |
| `meta-openembedded/meta-networking` | same | `networking-layer` | NetworkManager, Avahi |
| `meta-openembedded/meta-filesystems` | same | `filesystems-layer` | filesystem utilities |
| `meta-openembedded/meta-python` | same | `meta-python` | Python packages |
| `meta-virtualization` | `git.yoctoproject.org/meta-virtualization` | `virtualization-layer` | containerd, runc, nerdctl |
| `meta-raspberrypi` | `github.com/agherzan/meta-raspberrypi` | `raspberrypi` | RPi BSP (RPi builds only) |
| `meta-tegra` | `github.com/OE4T/meta-tegra` | `tegra` | NVIDIA Jetson BSP (Tegra builds only) |
| `meta-mender` | Mender project | `mender` | OTA update client (Orin tegra234 builds only) |
| `wendyos` (the repo itself) | local | `wendyos` | WendyOS distro and images |

### meta-wendyos-rpi (standalone RPi layer)

`LAYERDEPENDS_edgeos-rpi = "core raspberrypi virtualization-layer"`

`bblayers.conf` includes:

```
repos/meta-openembedded/meta-filesystems
repos/meta-openembedded/meta-multimedia
repos/meta-openembedded/meta-networking
repos/meta-openembedded/meta-oe
repos/meta-openembedded/meta-python
repos/meta-virtualization
repos/meta-raspberrypi
repos/poky/meta
repos/poky/meta-poky
repos/poky/meta-yocto-bsp
..   (the meta-wendyos-rpi layer itself)
```

### meta-wendyos-virtual (standalone VM layer)

`LAYERDEPENDS_edgeos-vm = "core virtualization-layer"`

`bblayers.conf` includes:

```
repos/meta-openembedded/meta-filesystems
repos/meta-openembedded/meta-networking
repos/meta-openembedded/meta-oe
repos/meta-openembedded/meta-python
repos/meta-virtualization
repos/poky/meta
repos/poky/meta-poky
repos/poky/meta-yocto-bsp
..   (the meta-wendyos-virtual layer itself)
```

Note: `meta-multimedia` and `meta-raspberrypi` are not included (no audio or RPi hardware needed in the VM layer).

---

## Layer Descriptions

### poky (meta + meta-poky + meta-yocto-bsp)

The upstream Yocto reference distribution. Provides the core OE classes, the `poky` distro defaults that WendyOS `require`s and then overrides, and reference BSPs for QEMU targets.

### meta-openembedded

A family of layers that extends OE-Core with recipes not included in Poky:

| Sub-layer | Key packages |
|---|---|
| `meta-oe` | Extended tooling, system utilities |
| `meta-networking` | NetworkManager, Avahi/mDNS, iptables, tcpdump |
| `meta-filesystems` | dosfstools, e2fsprogs, btrfs-tools |
| `meta-python` | Python libraries needed by recipes |
| `meta-multimedia` | GStreamer, PipeWire, WirePlumber, ALSA (RPi full images) |

### meta-virtualization

Provides the container runtime stack used by all WendyOS targets:

- `containerd-opencontainers` — the container runtime daemon
- `runc-opencontainers` — the OCI container runtime
- `nerdctl` — Docker-compatible CLI for containerd
- `cni` — Container Network Interface plugins

All WendyOS distros set `DISTRO_FEATURES:append = " virtualization"` to activate recipes from this layer.

### meta-raspberrypi

Raspberry Pi BSP layer. Provides:

- `linux-raspberrypi` — the RPi-patched kernel
- RPi firmware blobs (`raspberrypi-firmware`)
- BCM wireless and Bluetooth firmware packages
- RPi-specific device tree overlays and config.txt management

### meta-tegra (wendyos repo only)

NVIDIA Jetson BSP layer (OE4T project). Provides:

- L4T (Linux for Tegra) kernel and bootloader
- UEFI firmware packages
- Tegra-specific flash utilities (`tegraflash`)
- NVIDIA Container Toolkit support

The common meta-tegra paths (meta-tegra, meta-tegra-community, meta-tegra-extensions) are shared between Orin and Thor via `conf/template/include/bblayers/tegra-thor.inc`. The Orin (`tegra.inc`) build layers the Mender stack on top of that shared base; the Thor build uses `tegra-thor.inc` directly (no Mender in Phase 1).

### meta-mender (wendyos repo, Orin tegra234 targets only)

Adds Mender OTA update support:

- `mender` — the Mender client daemon
- A/B root filesystem partition scheme
- Mender artifact creation at image build time
- `MENDER_FEATURES_ENABLE` activates Mender via `mender-full` class

Mender is gated on `WENDYOS_MENDER`. It is enabled by default only for Orin (tegra234). RPi, QEMU, and Thor (tegra264) builds skip the `mender.inc` include fragment. Set `WENDYOS_MENDER = "1"` in `local.conf` to opt in on any target.

### WendyOS layer (meta-wendyos / meta-wendyos-rpi / meta-wendyos-virtual)

The top-level application layer. Provides:

- Distro configuration (`conf/distro/wendyos.conf`, `edgeos-rpi.conf`, `edgeos-vm.conf`)
- Machine configurations (see [configuration.md](configuration.md))
- Image recipes (`wendyos-image`, `edgeos-rpi-image`, `edgeos-vm-image`)
- Package group recipes (`packagegroup-edgeos-*`)
- `edgeos-agent` — downloads and installs the `wendy-agent` binary
- `edgeos-identity` — generates device UUID and friendly name on first boot
- `edgeos-motd` — login banner
- `systemd-mount-containerd` — systemd mount unit for the `/data` partition (containerd storage)
- WIC kickstart files defining disk partition layouts (located under `files/wic/`)

---

## Layer Priority

All WendyOS layers are registered with `BBFILE_PRIORITY = "50"`, which is higher than Poky's default (`5`) and meta-openembedded (`6`). This ensures WendyOS `.bbappend` files and overrides win over upstream defaults without needing explicit priority escalation per recipe.

---

## Adding a New Layer

1. Clone the layer into `repos/` (either by editing `bootstrap.sh` or manually).
2. Add its path to `build/conf/bblayers.conf` (or the appropriate include fragment under `conf/template/include/bblayers/`).
3. Verify the layer is compatible with the active series: `LAYERSERIES_COMPAT` in its `conf/layer.conf` must include `"scarthgap"` (for Orin/RPi/QEMU boards) or `"wrynose"` (for Thor).
4. Check its `LAYERDEPENDS` — ensure all upstream layers it depends on are already present.
5. Run `bitbake-layers show-layers` to confirm the layer is visible.

---

## Cloned Upstream Pinning

`bootstrap.sh` clones upstream layers at the appropriate branch tip for each board. Orin, RPi, and QEMU boards use the `scarthgap` branch; Thor uses `wrynose`. The sstate-cache is partitioned per layer tree (e.g. `sstate-cache/scarthgap/`, `sstate-cache/wrynose/`) so cross-series builds on the same workspace do not collide. Downloads are shared across series.

For reproducible builds you can pin specific commits by editing the `repos` array in `bootstrap.sh`, or by providing per-board overrides in `conf/template/boards/<board-id>/repos.overrides`.

The Thor board (`jetson-agx-thor`) pins the following layer trees in its `repos.overrides`:

| Layer | Branch / source |
|---|---|
| bitbake | master (2.19.0) |
| openembedded-core | wrynose HEAD |
| meta-yocto | wrynose HEAD |
| meta-openembedded | wrynose HEAD |
| meta-virtualization | master HEAD (no wrynose branch upstream) |
| meta-tegra | master-l4t-r38.4.x |
| meta-tegra-community | master-l4t-r38.4.x |
