# Meta-Layers

WendyOS is built from a set of stacked Yocto / OpenEmbedded meta-layers. Every build begins with the OpenEmbedded core and Poky distro policy and adds hardware BSP layers, then the WendyOS application layer on top.

The Yocto core is composed from three upstream split repos (`bitbake`, `openembedded-core`, `meta-yocto`) rather than a bundled `poky.git` monolith. All repos are cloned into a per-series tree namespace: `repos/<tree>/`, where `<tree>` is controlled by `WENDYOS_LAYER_TREE` (default `scarthgap`).

Layers declare compatibility with `LAYERSERIES_COMPAT_wendyos = "scarthgap wrynose"`. Orin, RPi, and QEMU boards build on **scarthgap**; AGX Thor builds on **wrynose**.

---

## Layers by Target

### wendyos (unified repo)

Layers are assembled per board by `bblayers.conf` fragments under `conf/template/include/bblayers/`. The full set for a Jetson + RPi + QEMU build is:

| Layer | Source | Collection Name | Purpose |
|---|---|---|---|
| `openembedded-core/meta` | `git.openembedded.org/openembedded-core` | `core` | OpenEmbedded core |
| `meta-yocto/meta-poky` | `git.yoctoproject.org/meta-yocto` | `yocto` | Poky distro policy |
| `meta-openembedded/meta-oe` | `github.com/openembedded/meta-openembedded` | `openembedded-layer` | Additional recipes |
| `meta-openembedded/meta-networking` | same | `networking-layer` | NetworkManager, Avahi |
| `meta-openembedded/meta-filesystems` | same | `filesystems-layer` | filesystem utilities |
| `meta-openembedded/meta-python` | same | `meta-python` | Python packages |
| `meta-virtualization` | `git.yoctoproject.org/meta-virtualization` | `virtualization-layer` | containerd, runc, nerdctl |
| `meta-raspberrypi` | `github.com/agherzan/meta-raspberrypi` | `raspberrypi` | RPi BSP (RPi builds only) |
| `meta-tegra` | `github.com/OE4T/meta-tegra` | `tegra` | NVIDIA Jetson BSP (Tegra builds only) |
| `meta-mender` | Mender project | `mender` | OTA update client (Orin tegra234 builds only) |
| `wendyos` (the repo itself) | local | `wendyos` | WendyOS distro and images |

All layer paths inside `bblayers.conf` are expressed as `${TOPDIR}/../repos/${WENDYOS_LAYER_TREE}/<layer>`. `bitbake` is sourced via `oe-init-build-env` from `repos/${WENDYOS_LAYER_TREE}/openembedded-core/` and is not added to `BBLAYERS` directly.

### meta-wendyos-rpi (standalone RPi layer)

`LAYERDEPENDS_edgeos-rpi = "core raspberrypi virtualization-layer"`

`bblayers.conf` includes:

```
repos/<tree>/meta-openembedded/meta-filesystems
repos/<tree>/meta-openembedded/meta-multimedia
repos/<tree>/meta-openembedded/meta-networking
repos/<tree>/meta-openembedded/meta-oe
repos/<tree>/meta-openembedded/meta-python
repos/<tree>/meta-virtualization
repos/<tree>/meta-raspberrypi
repos/<tree>/openembedded-core/meta
repos/<tree>/meta-yocto/meta-poky
..   (the meta-wendyos-rpi layer itself)
```

### meta-wendyos-virtual (standalone VM layer)

`LAYERDEPENDS_edgeos-vm = "core virtualization-layer"`

`bblayers.conf` includes:

```
repos/<tree>/meta-openembedded/meta-filesystems
repos/<tree>/meta-openembedded/meta-networking
repos/<tree>/meta-openembedded/meta-oe
repos/<tree>/meta-openembedded/meta-python
repos/<tree>/meta-virtualization
repos/<tree>/openembedded-core/meta
repos/<tree>/meta-yocto/meta-poky
..   (the meta-wendyos-virtual layer itself)
```

Note: `meta-multimedia` and `meta-raspberrypi` are not included (no audio or RPi hardware needed in the VM layer).

---

## Layer Descriptions

### openembedded-core (meta) and meta-yocto (meta-poky)

`openembedded-core` provides the OE core classes and the `core` collection. `meta-yocto` provides the `yocto` collection (Poky distro policy) that WendyOS `require`s and then overrides. `bitbake` is cloned as a sibling repo and activated via `oe-init-build-env` rather than added to `BBLAYERS`.

These three repos replace the former bundled `poky.git` monolith. Series branches through `walnascar` (5.2) were available in the monolith; `whinlatter` (5.3) and later ship exclusively as split repos.

### meta-openembedded

A family of layers that extends OE-Core with recipes not included in OE-Core:

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

All WendyOS layers are registered with `BBFILE_PRIORITY = "50"`, which is higher than OE-Core's default (`5`) and meta-openembedded (`6`). This ensures WendyOS `.bbappend` files and overrides win over upstream defaults without needing explicit priority escalation per recipe.

---

## Adding a New Layer

1. Clone the layer into `repos/<tree>/` (either by editing `bootstrap.sh` or manually).
2. Add its path to `build/conf/bblayers.conf` (or the appropriate include fragment under `conf/template/include/bblayers/`), using `${WENDYOS_LAYER_TREE}` in the path.
3. Verify the layer is compatible with the active series: `LAYERSERIES_COMPAT` in its `conf/layer.conf` must include the value of `WENDYOS_LAYER_TREE`.
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
