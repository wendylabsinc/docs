# Troubleshooting

Quick fixes for the most common issues.

---

## Device not showing up in `wendy discover`

**Check the network first.** Your laptop and device must be on the same WiFi network, or connected via the USB cable.

```sh
wendy discover --timeout 10s
```

If it still doesn't appear:

- **WiFi:** Confirm your laptop joined the hackathon network, not a personal hotspot. mDNS doesn't cross network boundaries.
- **Device not booted:** The LED should be solid white. If it's off or flashing, the device needs more time or is still booting for the first time.
- **macOS Local Network permission:** macOS may have blocked mDNS. Go to **System Settings → Privacy & Security → Local Network** and make sure Terminal (or `wendy`) is allowed.

---

## `wendy run` fails with "no device selected"

You haven't set a default device. Either pass one explicitly:

```sh
wendy run --device my-jetson
```

Or set a default once and forget about it:

```sh
wendy device set-default my-jetson
```

---

## Build fails: "Cannot connect to the Docker daemon"

Docker Desktop isn't running. Start it and wait for the whale icon in the menu bar, then retry. On Apple silicon, you can skip Docker entirely:

```sh
wendy run --builder apple-container
```

---

## Build fails: "exec format error" or wrong architecture

Your container was built for the wrong CPU. `wendy run` cross-compiles automatically, but Docker buildx must be set up for multi-platform builds:

```sh
docker buildx create --use
```

Then retry `wendy run`.

---

## App starts but I can't reach it in the browser

**1. Bind to `0.0.0.0`, not `localhost`.**

Your app must listen on all interfaces. In Python/FastAPI:

```python
uvicorn.run(app, host="0.0.0.0", port=8000)
```

**2. Add the network entitlement.**

`wendy.json` needs `"mode": "host"` so the container shares the device's network:

```json
{ "type": "network", "mode": "host" }
```

**3. Find the device hostname.**

```sh
wendy discover
```

The `ADDRESS` column shows the hostname, e.g. `my-jetson.local`. Open `http://my-jetson.local:8000`.

**4. Check what's running.**

```sh
wendy device apps list
wendy device logs
```

---

## GPU not available inside the container

Add the `gpu` entitlement to `wendy.json` — the GPU is not accessible by default:

```json
{
  "entitlements": [
    { "type": "gpu" }
  ]
}
```

Redeploy with `wendy run`. To confirm GPU access inside the container:

```sh
wendy device logs
```

Look for `CUDA Available: True` or equivalent output from your framework.

---

## First deploy is slow

Normal. The first `wendy run` pulls a base image, installs dependencies, and pushes the full container to the device. Subsequent deploys use chunk-diff transfer — only changed bytes are sent — and typically finish in under 10 seconds for a Python code change.

---

## `wendy mcp setup` says "No supported AI tools detected"

The command looks for known config directories or binaries on `PATH`. If you installed Claude Code recently:

```sh
which claude      # confirm it's on PATH
wendy mcp setup   # re-run
```

For a manual workaround, see [Manual setup](../wendy-agent/mcp.md#manual).

---

## Certificate expired or auth errors

Check the state first:

```sh
wendy auth status
```

Refresh an expired certificate:

```sh
wendy auth refresh-certs
```

If that fails, log in again:

```sh
wendy auth logout
wendy auth login
```

---

## Container keeps restarting

```sh
wendy device logs
```

The restart reason is usually a crash in your app code, a missing dependency, or a port already in use from a previous deploy. Fix the code and run `wendy run` again — the old container is replaced automatically.
