Installs WendyOS onto an NVME or SD card.

You can provide it with an image path (pointing to an `.img` file), and drive to flash to. If not provided, will provide an interactive guided experience.

Can be called with `--nightly` to list and install nightly WendyOS releases.

Lists all supported WendyOS versions from a [GCP bucket](../../../../operations/gcp.md), where we host a JSON manifest. The format of these files is created by [Wendy OS Publisher](../../../../wendy-os-publisher/).

This tool supports both NVME and SD flashing of [WendyOS](../../../../wendyos/) (using `dd`), as well as [Wendy Lite](../../../../wendy-lite/) (ESP32c6).

1. Lists all available OS releases
2. Upon selection, lists available disks (or microcontrollers) to flash to.
3. [Downloads the file](./download.md) to local developer machine [cache](./cache.md)
4. Flashes the device

> **TODO**: Post-flashing, the device needs [provisioning](../device/setup.md). This includes updating the latest [`wendy-agent`](../../../../wendy-agent/) on Linux, provisioning a [certificate](../../../../pki/) and [Wendy Cloud](../../../../cloud/)