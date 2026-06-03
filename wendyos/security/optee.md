# OP-TEE Trusted Execution Environment

All Tegra (Jetson) WendyOS images ship an OP-TEE stack that provides a hardware-backed secure storage and cryptographic key store. QEMU and Raspberry Pi images do not include OP-TEE.

---

## Components

### Installed on every Tegra image

| Package | Description |
|---------|-------------|
| `optee-client` | Normal-world userspace. Ships `tee-supplicant`, the Linux daemon that backs REE-FS secure storage by writing TEE-encrypted blobs to `/var/lib/tee`, loads Trusted Applications (TAs) on demand, and forwards RPMB requests. |
| `optee-os` | Filesystem TAs installed to `/lib/optee_armtz`, including the PKCS#11 TA. `tee-supplicant` loads these from REE-FS at runtime; no QSPI/firmware reflash is needed to add or update them. |
| `optee-osdev` | Trusted OS core (`tee.bin`/`tee.elf`). Flashed to QSPI by the bootloader; not part of the rootfs. |
| `systemd-mount-tee` | Bind-mounts `/data/tee` onto `/var/lib/tee` so OP-TEE secure storage survives Mender A/B OTA updates (see [Persistent storage](#persistent-storage-across-ota-updates)). |

### Installed on Tegra debug images only (`WENDYOS_DEBUG=1`)

| Package | Description |
|---------|-------------|
| `optee-test` | `xtest`, the OP-TEE conformance suite including PKCS#11 regression tests. |
| `opensc` | `pkcs11-tool`, for driving the PKCS#11 token through the standard Cryptoki module `/usr/lib/libckteec.so.0`. |

---

## PKCS#11 Trusted Application

The PKCS#11 TA (UUID `fd02c9da-306c-48c7-a49c-bbd827ae86ee`) is built with `CFG_PKCS11_TA=y` and implements the GlobalPlatform Cryptoki interface. It provides a token for storing device keys — such as `device-tls` and UEFI PK/KEK — in OP-TEE secure storage instead of in cleartext on the rootfs.

The TA is delivered as part of the `optee-os` package to `/lib/optee_armtz`. `tee-supplicant` loads it from REE-FS on demand at runtime; no firmware reflash is required to add it.

> **Security note:** On a non-fused board, secure storage roots in the OP-TEE test key. Keys stored in the PKCS#11 token are only genuinely hardware-bound after `OEM_K1` is burned.

---

## Persistent storage across OTA updates

`tee-supplicant` writes TEE-encrypted blobs to `/var/lib/tee`. By default that path is on the rootfs, which Mender A/B OTA replaces on every update — destroying any stored PKCS#11 tokens and device keys.

The `systemd-mount-tee` package installs the `var-lib-tee.mount` unit, which bind-mounts `/data/tee` onto `/var/lib/tee` before `tee-supplicant.service` starts:

```ini
[Mount]
What=/data/tee
Where=/var/lib/tee
Type=none
Options=bind,nosuid,nodev,x-systemd.mkdir
```

The `/data` partition sits outside the Mender A/B update boundary, so PKCS#11 tokens and device keys survive OTA updates. The unit carries `RequiresMountsFor=/data`, so it is inert on any configuration that lacks a `/data` partition.

This follows the same pattern as the `systemd-mount-containerd` unit that backs `/var/lib/containerd` from `/data/containerd`.

---

## Runtime dependencies

OP-TEE requires:

- The OP-TEE OS (`tee.bin`) loaded by the bootloader from QSPI.
- The `tee` kernel driver functional at runtime.

Both are provided by the Tegra boot assembly and are present on all supported Jetson hardware. `tee-supplicant` must be running for any TA that uses persistent objects (PKCS#11 tokens, device keys) to have a backing store.

---

## Validating OP-TEE on a debug image

With `optee-test` and `opensc` installed (debug images only):

```sh
# Run the full OP-TEE conformance suite (includes PKCS#11 tests)
xtest

# Run only PKCS#11 tests
xtest -t pkcs11

# List tokens via the Cryptoki module
pkcs11-tool --module /usr/lib/libckteec.so.0 --list-slots

# Inspect objects in the token
pkcs11-tool --module /usr/lib/libckteec.so.0 --list-objects
```
