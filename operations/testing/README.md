# CI Testing

The `scripts/test-ci.sh` script runs integration and hardware-in-the-loop tests against a real WendyOS device.

## Usage

```sh
scripts/test-ci.sh [-t <test-name>] [-h <device-hostname>]
```

If `--hostname` is not provided, the script auto-discovers a device on the local network.

Use `-t` one or more times to run specific tests. Omit `-t` to run all tests.

## Available tests

| Test name | Description |
|-----------|-------------|
| `python-no-bluetooth` | Verify bluetooth is blocked WITHOUT entitlement |
| `compose-hello` | docker-compose multi-service deployment with build: Dockerfiles |
| `compose-images` | docker-compose multi-service deployment using public images |
| `otel-localhost-only` | Verify OTEL receivers (4317/4318) are not reachable from the network |

### `otel-localhost-only`

Verifies the security property introduced in PR #597 (WDY-1097, WDY-1100): both OTEL receivers must be bound to `127.0.0.1` only and must not accept connections from external network interfaces.

Two sub-checks are run against the target device:

| Sub-test | Port | What is checked |
|----------|------|-----------------|
| `otel-localhost-only (gRPC 4317 not reachable)` | 4317 | `nc -z -w 3 <device> 4317` must fail |
| `otel-localhost-only (HTTP 4318 not reachable)` | 4318 | `nc -z -w 3 <device> 4318` must fail |

Run it with:

```sh
scripts/test-ci.sh -t otel-localhost-only -h <device-hostname-or-ip>
```

See [wendy-agent/otel.md](../../wendy-agent/otel.md) for full OTEL receiver documentation and the [Threat Model](../../security/THREAT_MODEL.md) for the security context.
