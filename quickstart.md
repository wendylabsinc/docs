# Quickstart

From nothing to a running app on your Jetson or Raspberry Pi in about five minutes.

## 1. Install the CLI

**macOS (Homebrew):**

```sh
brew install wendylabsinc/tap/wendy
```

**Linux / Windows:**

```sh
curl -fsSL https://wendy.sh/install.sh | bash
```

## 2. Find your device

Your device is either connected via USB or on the same WiFi network. Either way:

```sh
wendy discover
```

You'll see something like:

```
NAME               ADDRESS                  SOURCE
my-jetson          my-jetson.local:50051    mDNS
```

Save a round of typing on every command by setting a default:

```sh
wendy device set-default my-jetson
```

> **Device not showing up?** Make sure your laptop and device are on the same network. For USB, the wendy USB network adapter shows up automatically — no driver install needed. If you still don't see it, ask an organizer: the device may need to be plugged into power first.

## 3. Deploy your first app

Create a new directory and three files:

**`main.py`**

```python
from fastapi import FastAPI
import os

app = FastAPI()

@app.get("/")
def hello():
    return {"message": "Hello from my Jetson!", "hostname": os.uname().nodename}
```

**`requirements.txt`**

```
fastapi>=0.109.0
uvicorn[standard]>=0.27.0
```

**`wendy.json`**

```json
{
  "appId": "com.example.hello",
  "version": "1.0.0",
  "language": "python",
  "entitlements": [{ "type": "network", "mode": "host" }],
  "hooks": {
    "postStart": {
      "openURL": "http://${WENDY_HOSTNAME}:8000"
    }
  }
}
```

Deploy it:

```sh
wendy run
```

`wendy run` detects `requirements.txt`, builds the container for your device's architecture (arm64), pushes it, and starts it. The browser opens automatically when the app is ready. The whole thing takes about 30 seconds on the first run; subsequent deploys are much faster because only changed files are transferred.

## 4. Connect your AI assistant

Run this once to wire `wendy mcp serve` into Claude Code, Cursor, Windsurf, or whatever you have installed:

```sh
wendy mcp setup
```

Restart your AI tool. From now on it has direct access to your device — it can deploy code, read logs, check hardware stats, and manage WiFi without leaving the editor.

Call `wendy_status` at the start of any AI session to confirm the connection and get a suggested next step.

## 5. Iterate

Edit `main.py`, save. `wendy watch` redeploys automatically on every save:

```sh
wendy watch
```

No manual `wendy run` needed while you're building. Press `Ctrl-C` to stop watching.

---

## Common next steps

| Goal | Command |
|------|---------|
| Tail app logs | `wendy device logs` |
| See running containers | `wendy device apps list` |
| Check device CPU / memory | `wendy device dashboard` |
| Access the GPU | Add `{ "type": "gpu" }` to `entitlements` in `wendy.json` |
| Connect to a camera | Add `{ "type": "camera" }` to `entitlements` |
| Persist data across restarts | Add `{ "type": "persist", "name": "my-app", "path": "/data" }` |

The full reference for `wendy.json` fields and entitlements is at [apps/wendy.json](./apps/wendy.json.md).
