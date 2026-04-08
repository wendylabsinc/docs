Wendy CLI has a few global flags that are __always_ visible:

## `--json`

This flag configures (most) commands to format their output as JSON, so it can be parsed and used in scripts and tools.

## `--device`

Overrides the [default device](device-selection.md), connecting to the speicifed hostname or IP.

## TODO: `--insecure`

Allows the Wendy-Agent to connect to unprovisioned devices without verified TLS.

> **TODO (urgent)**: This flag is not exposed yet.