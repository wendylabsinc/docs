# Data Partition Initialisation

`wendyos-data-init.service` runs before `data.mount` on every boot and ensures the `/data` partition holds a usable ext4 filesystem.

## What it does

1. Resolves the partition device by GPT label `data`.
2. Checks whether the partition is already initialised (see [Idempotency guard](#idempotency-guard) below).
3. If not initialised: fixes the GPT backup header (the flashed image is smaller than the physical disk), grows the partition to fill the disk, and formats it with `mkfs.ext4`.

## Idempotency guard

The service treats a partition as already initialised only when **both** of the following are true:

- `blkid` reports the partition type as `ext4`.
- The filesystem physically fits the partition: `fs_block_count × block_size ≤ device_size_bytes`.

The size check is necessary because a reflash recreates the data partition at its small initial size (for example, 512 MiB) while leaving behind the superblock of a previously-grown filesystem (which may have been ~900 GB). `blkid` still reports `ext4` in this situation, but the kernel refuses to mount the filesystem:

```
EXT4-fs (nvme0n1p17): bad geometry: block count 236275281 exceeds size of device (131072 blocks)
```

When an ext4 superblock is present but does not fit the current partition, the guard logs a message and falls through to grow + `mkfs.ext4`, which overwrites the stale superblock.

The check keys on the filesystem itself rather than a per-rootfs stamp, so it is A/B-slot-safe.

## Impact of a failed mount

If `data.mount` fails, the bind mounts that depend on `/data` (`home`, `var-log`, `containerd`) are also unavailable, which cascades to `user@1000` and related services such as pipewire. The idempotency guard prevents this failure mode after a reflash.
