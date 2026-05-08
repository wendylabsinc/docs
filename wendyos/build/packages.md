# Package Management

WendyOS uses RPM packages internally (`PACKAGE_CLASSES = "package_rpm"`). You do not install or upgrade packages with rpm or dnf at runtime — all packages are built from Yocto recipes at image build time and baked into the WIC image.

---

## Adding Packages to an Image

### Via IMAGE_INSTALL in the image recipe

The most direct method. In your image `.bb` or `.bbappend`:

```bitbake
IMAGE_INSTALL:append = " \
    curl \
    jq \
    python3 \
    python3-requests \
    "
```

The space before the first package name is required when using `:append`.

### Via a package group

Group related packages together and include the group in the image:

```bitbake
# recipes-core/packagegroups/packagegroup-my-feature.bb
inherit packagegroup

RDEPENDS:${PN} = " \
    curl \
    jq \
    python3 \
    "
```

```bitbake
# In the image recipe:
IMAGE_INSTALL:append = " packagegroup-my-feature"
```

### Via IMAGE_FEATURES

Some package sets are pre-defined as image features:

```bitbake
IMAGE_FEATURES += "ssh-server-openssh"   # openssh-sshd
IMAGE_FEATURES += "tools-debug"           # gdb, strace, ltrace
IMAGE_FEATURES += "dev-pkgs"             # -dev packages for all installed recipes
IMAGE_FEATURES += "tools-profile"        # oprofile, lttng, valgrind
IMAGE_FEATURES += "tools-testapps"       # test tools
```

---

## Package Name Mapping

Yocto package names differ from the upstream project names. These are the most commonly needed mappings for WendyOS:

| What you want | Yocto package name | Recipe |
|---|---|---|
| containerd | `containerd-opencontainers` | meta-virtualization |
| runc | `runc-opencontainers` | meta-virtualization |
| nerdctl | `nerdctl` | meta-virtualization |
| CNI plugins | `cni` | meta-virtualization |
| NetworkManager | `networkmanager` | meta-networking |
| nmcli | `networkmanager-nmcli` | meta-networking |
| Avahi daemon | `avahi-daemon` | meta-networking |
| avahi-browse etc | `avahi-utils` | meta-networking |
| htop | `htop` | meta-oe |
| vim | `vim` | meta-oe |
| strace | `strace` | meta-oe |
| tcpdump | `tcpdump` | meta-networking |
| iperf3 | `iperf3` | meta-networking |
| i2c-tools | `i2c-tools` | meta-oe |
| SPI tools | `spitools` | meta-oe |
| curl | `curl` | meta-oe |
| ca-certificates | `ca-certificates` | meta-oe |
| jq | `jq` | meta-oe |
| Python 3 | `python3` | poky |
| pip | `python3-pip` | poky |
| rsync | `rsync` | meta-oe |
| openssh | `openssh` | poky |
| openssh server | `openssh-sshd` | poky |
| openssh SFTP | `openssh-sftp-server` | poky |
| GStreamer core + base plugins | `gstreamer1.0-plugins-base` | meta-multimedia |
| GStreamer good plugins (VP8/vp8enc, etc.) | `gstreamer1.0-plugins-good` | meta-multimedia |
| GStreamer bad plugins | `gstreamer1.0-plugins-bad` | meta-multimedia |
| GStreamer ugly plugins (H.264/x264enc) | `gstreamer1.0-plugins-ugly` | meta-multimedia |
| GStreamer libav (FFmpeg) | `gstreamer1.0-libav` | meta-multimedia |
| GStreamer NVIDIA V4L2 plugins (Tegra) | `gstreamer1.0-plugins-nvvideo4linux2` | meta-tegra |

To search for a package name from a recipe name:

```bash
bitbake -e <recipe-name> | grep ^PACKAGES
```

---

## GStreamer PACKAGECONFIG Customizations

WendyOS applies the following PACKAGECONFIG overrides to the GStreamer plugin packages:

| Package | Change | Reason |
|---|---|---|
| `gstreamer1.0-plugins-good` | `vpx` enabled | Provides `vp8enc` + `webmmux` for VP8 video streaming fallback |
| `gstreamer1.0-plugins-ugly` | `x264` enabled | Provides `x264enc` (H.264 software encoder) for video streaming |
| `gstreamer1.0-plugins-bad` | `vulkan` disabled | WendyOS disables both x11 and Wayland; `vulkansink` requires a windowing system and causes a configure-time error |

The x264 encoder carries a commercial license flag. WendyOS accepts it globally via `LICENSE_FLAGS_ACCEPTED += "commercial"` in `wendyos.conf`.

---

## Writing a Custom Recipe

### Minimal recipe for a binary

```bitbake
# recipes-myapp/myapp/myapp_1.0.bb

SUMMARY = "My application"
DESCRIPTION = "Does something useful on WendyOS"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

SRC_URI = "https://example.com/myapp-${PV}.tar.gz"
SRC_URI[sha256sum] = "<sha256-of-tarball>"

S = "${WORKDIR}/myapp-${PV}"

do_compile() {
    make
}

do_install() {
    install -d ${D}${bindir}
    install -m 0755 myapp ${D}${bindir}/myapp
}
```

### Recipe with RDEPENDS (runtime dependencies)

```bitbake
RDEPENDS:${PN} = "bash curl python3"
```

`RDEPENDS` are runtime dependencies — packages that must be installed alongside yours. Use this for interpreters, shared libraries, or tools your binary calls at runtime.

`DEPENDS` (no `R`) is for build-time dependencies — tools or headers needed to compile your recipe, but not needed on the target.

### Recipe that installs a pre-built binary (like edgeos-agent)

The `edgeos-agent` recipe downloads the `wendy-agent` binary from GitHub at build time and installs it. This pattern requires allowing network access:

```bitbake
do_compile[network] = "1"

do_compile() {
    wget -O ${B}/myapp "https://releases.example.com/myapp-${PV}-aarch64"
    chmod +x ${B}/myapp
}

do_install() {
    install -d ${D}${bindir}
    install -m 0755 ${B}/myapp ${D}${bindir}/myapp
}

# Suppress QA checks for pre-built binaries
INSANE_SKIP:${PN} += "already-stripped"
```

### Recipe with a systemd service

```bitbake
inherit systemd

SRC_URI += "file://myapp.service"

SYSTEMD_SERVICE:${PN} = "myapp.service"
SYSTEMD_AUTO_ENABLE:${PN} = "enable"

do_install:append() {
    install -d ${D}${systemd_system_unitdir}
    install -m 0644 ${WORKDIR}/myapp.service ${D}${systemd_system_unitdir}/
}

FILES:${PN} += "${systemd_system_unitdir}/myapp.service"
```

---

## Using .bbappend to Modify Existing Recipes

A `.bbappend` file extends an existing recipe without forking it. The filename must match the recipe it targets (with `%` as a wildcard for the version):

```bitbake
# recipes-multimedia/gstreamer/gstreamer1.0_%.bbappend
# (as used in meta-wendyos-rpi to customise GStreamer build flags)

PACKAGECONFIG:append = " gst-tracer-hooks"
```

BitBake applies `.bbappend` files in layer priority order. Since WendyOS layers have `BBFILE_PRIORITY = "50"`, their appends win over Poky (`5`) and meta-openembedded (`6`).

---

## Removing Packages from an Image

### Remove from IMAGE_INSTALL

```bitbake
IMAGE_INSTALL:remove = "package-name"
```

### Remove an IMAGE_FEATURE

```bitbake
IMAGE_FEATURES:remove = "ssh-server-openssh"
```

### Prevent a package from being pulled in as a recommendation

Some packages are added via `RRECOMMENDS` (weak dependencies). To block them:

```bitbake
BAD_RECOMMENDATIONS += "package-name"
```

### Prevent a package from ever being installed

```bitbake
PACKAGE_EXCLUDE = "package-name"
```

---

## Inspecting Package Dependencies

```bash
# Show what a recipe will produce
bitbake -e <recipe> | grep ^PACKAGES

# Show the full dependency graph for an image
bitbake -g wendyos-image
# Creates task-depends.dot — view with graphviz: dot -Tsvg task-depends.dot -o deps.svg

# Show why a package is included
bitbake-layers show-recipes <package-name>

# Find which layer provides a recipe
bitbake-layers find-recipe <recipe-name>
```

---

## Installed Package Manifest

After every build, BitBake writes a manifest of every installed package:

```
build/tmp/deploy/images/<machine>/wendyos-image-<machine>.rootfs.manifest
```

Each line is `<package-name> <architecture> <version>`. Use this to audit what is in your image and to track changes between builds.

---

## devtool: Iterating on a Recipe Quickly

`devtool` is a BitBake utility for modifying recipes interactively without editing files by hand:

```bash
# Fetch source and set up a workspace recipe
devtool modify myapp

# Make your changes in the workspace
# Then rebuild
devtool build myapp

# Build a test image with the workspace version
devtool build-image wendyos-image

# When done, update the recipe in-tree
devtool finish myapp <layer-path>
```

`devtool` is included in the Poky SDK and in the Docker build container.
