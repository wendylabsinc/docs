Installs WendyOS onto an NVME or SD card.

You can provide it with an image path (pointing to an `.img` file), and drive to flash to. If not provided, will provide an interactive guided experience.

Can be called with `--nightly` to list and install nightly WendyOS releases.

Lists all supported WendyOS versions from a [GCP bucket](../../../../operations/gcp.md), where we host a JSON manifest. The format of these files is created by [Wendy OS Publisher](../../../../wendy-os-publisher/).

This tool supports both NVME and SD flashing of [WendyOS](../../../../wendyos/) (using `dd`), as well as [Wendy Lite](../../../../wendy-lite/) (ESP32c6).

1. Resolves the OS version — uses `--version` if provided, otherwise defaults to the latest release (or the latest nightly if `--nightly` is passed).
2. Upon selection, lists available disks (or microcontrollers) to flash to.
3. [Downloads the file](./download.md) to local developer machine [cache](./cache.md)
4. Flashes the device
5. Writes the [config partition](../../../../wendyos/config-partition.md) with the latest `wendy-agent` binary and, if WiFi credentials were provided, a `wendy.conf` — so the device connects to WiFi and self-updates the agent on first boot without any manual SSH access.

## WiFi pre-configuration

Pass `--wifi-ssid` to pre-configure WiFi. If only `--wifi-ssid` is given (no `--wifi-password`), the CLI checks the system keychain (macOS) and, if not found, prompts for the password interactively.

```sh
wendy os install --wifi-ssid MyNetwork --wifi-password hunter2
```

When running interactively (stdin is a terminal) without flags, the CLI asks whether to configure WiFi and offers to scan for nearby networks and look up the password from the system keychain.

> **TODO**: Post-flashing, the device still needs certificate provisioning and Wendy Cloud enrollment. See [device setup](../device/setup.md), [PKI](../../../../pki/), and [Wendy Cloud](../../../../cloud/).
