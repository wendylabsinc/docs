# Deploying to Wendy Lite

Wendy Lite devices are managed through the Wendy CLI using the WendyCom protocol, which runs over TLS with protobuf (nanopb) framing.

## Firmware Install and Provisioning

During firmware install, the Wendy CLI provisions the device with:

- **Wi-Fi credentials** — SSID and password the device uses to join the network.
- **mTLS and cloud certificates** — the device certificate, private key, and CA chain used for mutual TLS communication.

These values are written to a dedicated provisioning partition on the device flash at install time.

```sh
wendy lite install --wifi-ssid MyNetwork --wifi-password secret [firmware.bin]
```

## Connecting to a Device

After provisioning, the CLI establishes a mutual TLS (mTLS) connection to the Wendy Lite device using the certificates provisioned during install. Communication uses the WendyCom protocol (TLS + protobuf).

## Deploying a WASM App

Once connected, use the CLI to push a WASM app to the device:

```sh
wendy lite deploy [app.wasm]
```

The CLI transfers the WASM binary over the mTLS connection.

## Starting and Stopping the App

```sh
wendy lite start
wendy lite stop
```

These commands instruct the device to start or stop the currently installed WASM app over the mTLS connection.

## Known Limitations

> **Security:** The following certificate validation checks are not yet enforced:
>
> - The CLI does not verify that the CLI user's organisation matches the device's organisation in the TLS certificate.
> - Extended Key Usage (EKU) verification is disabled.
>
> These gaps will be addressed in a future release.
