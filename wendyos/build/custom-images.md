# Custom Images

This guide shows how to create a custom WendyOS image recipe, add packages, override distro defaults, and customise the disk partition layout.

---

## Understanding the Image Recipe Structure

The existing WendyOS image recipes are a good starting point:

- `meta-wendyos-rpi/recipes-core/images/edgeos-rpi-image.bb` — RPi image
- `meta-wendyos-virtual/recipes-core/images/edgeos-vm-image.bb` — VM image
- `wendyos/recipes-core/images/wendyos-image.bb` — unified image

A minimal image recipe:

```bitbake
DESCRIPTION = "My custom WendyOS image"
LICENSE = "MIT"

inherit core-image

# Force systemd as the init manager
DISTRO_FEATURES:append = " systemd"
VIRTUAL-RUNTIME_init_manager = "systemd"

# Output formats
IMAGE_FSTYPES += "wic wic.bmap"

# Packages to include
IMAGE_INSTALL:append = " \
    packagegroup-edgeos-rpi-base \
    edgeos-agent \
    edgeos-identity \
    "

# Minimum rootfs size in kB (content expands this automatically)
IMAGE_ROOTFS_SIZE ?= "8192"
```

---

## Creating a Custom Image Recipe

### Step 1: Create the recipe file

Place your recipe inside the WendyOS layer (or inside your own layer appended on top):

```
recipes-core/images/my-wendyos-image.bb
```

### Step 2: Choose a base class

```bitbake
inherit core-image
```

`core-image` is the standard class. It provides the `IMAGE_FEATURES` variable and wires up `IMAGE_INSTALL` automatically. All WendyOS images use it.

### Step 3: Add IMAGE_FEATURES

`IMAGE_FEATURES` controls pre-defined feature bundles:

```bitbake
IMAGE_FEATURES += "ssh-server-openssh"   # include OpenSSH
IMAGE_FEATURES += "debug-tweaks"          # empty root password, allow root SSH
IMAGE_FEATURES += "dev-pkgs"             # add -dev packages for every installed package
IMAGE_FEATURES += "tools-debug"          # gdb, strace, etc.
```

The production RPi image does NOT include `ssh-server-openssh` or `debug-tweaks` by default. The VM image includes both.

### Step 4: Add packages

```bitbake
IMAGE_INSTALL:append = " \
    my-custom-recipe \
    python3 \
    python3-pip \
    "
```

See [packages.md](packages.md) for how to write and include individual package recipes.

### Step 5: Set IMAGE_FSTYPES

```bitbake
# For SD card / NVMe (RPi)
IMAGE_FSTYPES += "wic wic.bmap wic.bz2"

# For VMs
IMAGE_FSTYPES += "wic wic.qcow2 wic.vmdk"

# For Jetson (tegraflash is added automatically by meta-tegra for Tegra machines)
IMAGE_FSTYPES += "ext4"
```

### Step 6: Build

```bash
bitbake my-wendyos-image
```

---

## Using Package Groups

Package groups are a way to name a collection of packages and include them as a unit. The WendyOS layers ship these pre-defined groups:

| Package group | Layer | Contents |
|---|---|---|
| `packagegroup-edgeos-rpi-base` | meta-wendyos-rpi | containerd, NetworkManager, Avahi, nerdctl, bash, coreutils, iproute2 |
| `packagegroup-edgeos-rpi-debug` | meta-wendyos-rpi | htop, vim, strace, tcpdump, iperf3, openssh, tree |
| `packagegroup-edgeos-vm-base` | meta-wendyos-virtual | containerd, NetworkManager, Avahi, nerdctl, curl, ca-certificates, htop, vim |
| `packagegroup-edgeos-vm-debug` | meta-wendyos-virtual | strace, tcpdump, iptables, procps, e2fsprogs, dosfstools |

Create your own package group recipe:

```bitbake
# recipes-core/packagegroups/packagegroup-my-app.bb

SUMMARY = "My application package group"
inherit packagegroup

RDEPENDS:${PN} = " \
    python3 \
    python3-requests \
    curl \
    jq \
    "
```

Then include it in your image:

```bitbake
IMAGE_INSTALL:append = " packagegroup-my-app"
```

---

## Overriding Distro Defaults

### Enabling SSH in a production image

```bitbake
IMAGE_FEATURES += "ssh-server-openssh"
```

### Enabling debug tweaks (empty root password)

Only for development images:

```bitbake
IMAGE_FEATURES += "debug-tweaks"
```

### Adding a custom user

The `wendyos-user` recipe (in `wendyos/recipes-extended/wendyos-user/`) creates the `wendy` user. You can create a similar recipe for your own user:

```bitbake
# recipes-extended/my-user/my-user_1.0.bb
SUMMARY = "Create operator user account"
inherit useradd

USERADD_PACKAGES = "${PN}"
USERADD_PARAM:${PN} = "-m -s /bin/bash -G sudo myuser"
```

### Customising the MOTD

Override `edgeos-motd` in your layer:

```bitbake
# recipes-core/edgeos-motd/edgeos-motd_1.0.bbappend
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"
# place your custom motd file in recipes-core/edgeos-motd/files/motd
```

### Overriding distro configuration in an image recipe

```bitbake
# Override VOLATILE_LOG_DIR for this image (persistent logs)
EDGEOS_PERSIST_JOURNAL_LOGS = "1"

# Or with the wendyos repo naming:
WENDYOS_PERSIST_JOURNAL_LOGS = "1"
```

---

## Customising the Partition Layout

WIC (Wic Image Creator) kickstart files (`.wks`) define the disk partition layout. The WendyOS layers ship:

| File | Used by | Layout |
|---|---|---|
| `wic/wendyos-rpi.wks` | RPi machines | MBR: 256 MB FAT32 `/boot`, 4 GB ext4 `/`, 2 GB ext4 `/data` |
| `wic/wendyos-vm.wks` | VM machine | GPT: 256 MB EFI `/boot`, 8 GB ext4 `/`, 2 GB ext4 `/data` |
| `wic/rpi-partuuid.wks` | `raspberrypi5-wendyos` (wendyos repo) | GPT with PARTUUID-based fstab |
| `wic/rpi-nvme-partuuid.wks` | `raspberrypi5-nvme-wendyos` | GPT, NVMe target |

To use a custom partition layout:

1. Create your `.wks` file:

   ```
   # my-custom.wks
   part /boot --source bootimg-partition --ondisk mmcblk0 --fstype=vfat --label boot --active --align 4096 --size 512
   part / --source rootfs --ondisk mmcblk0 --fstype=ext4 --label root --align 4096 --size 8192
   part /data --ondisk mmcblk0 --fstype=ext4 --label data --align 4096 --size 4096
   bootloader --ptable msdos
   ```

2. Set `WKS_FILE` in your machine conf or in `local.conf`:

   ```bitbake
   WKS_FILE = "my-custom.wks"
   ```

3. Place the file where BitBake can find it — either in `wic/` inside your layer, or added via `WKS_FILE_DEPENDS`.

---

## Extending Existing Image Recipes with .bbappend

If you want to add packages to the existing `edgeos-rpi-image` without forking it, create a `.bbappend` file in your layer:

```bitbake
# recipes-core/images/edgeos-rpi-image.bbappend

IMAGE_INSTALL:append = " \
    my-custom-recipe \
    htop \
    "

# Add SSH to a production image temporarily
IMAGE_FEATURES += "ssh-server-openssh"
```

The `.bbappend` must be in a layer that is listed in `bblayers.conf` and has a higher or equal `BBFILE_PRIORITY` than the layer containing the original recipe.

---

## Image Version Suffix

All WendyOS images set a release version via:

```bitbake
IMAGE_VERSION_SUFFIX ?= "${DISTRO_VERSION}"
```

This means artifact filenames contain the distro version string. You can override it in `local.conf`:

```bitbake
IMAGE_VERSION_SUFFIX = "myapp-1.2.3"
```

---

## Checking the Final Package List

After a successful build, inspect what ended up in the image:

```bash
# Inside the build environment:
bitbake -e my-wendyos-image | grep ^IMAGE_INSTALL

# Or check the manifest file generated at build time:
cat build/tmp/deploy/images/<machine>/my-wendyos-image-<machine>.rootfs.manifest
```

The manifest lists every installed package and its version.
