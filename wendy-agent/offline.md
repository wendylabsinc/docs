# Offline Installation

This page covers installing `wendy-agent` and related components on devices that do not have internet access, and working with air-gapped container images.

---

## Offline agent binary installation

When the device has no package manager and no internet access, download the release tarball on a connected machine and transfer it to the device.

### 1. Download on a connected machine

```sh
VERSION=v0.5.0
ARCH=arm64   # or amd64

curl -fsSL \
  "https://github.com/wendylabsinc/wendy-agent/releases/download/${VERSION}/wendy-agent-linux-${ARCH}-${VERSION#v}.tar.gz" \
  -o "wendy-agent-linux-${ARCH}.tar.gz"
```

### 2. Transfer to the device

```sh
scp "wendy-agent-linux-${ARCH}.tar.gz" user@device:/tmp/
```

### 3. Extract and install on the device

```sh
cd /tmp
tar -xzf wendy-agent-linux-${ARCH}.tar.gz
sudo install -m 755 wendy-agent-linux-${ARCH}/wendy-agent /usr/local/bin/wendy-agent
```

### 4. Set up the systemd service manually

```sh
sudo mkdir -p /var/lib/wendy-agent/storage /etc/wendy-agent

# Write the systemd unit
sudo tee /usr/lib/systemd/system/wendy-agent.service <<'EOF'
[Unit]
Description=Wendy Agent
After=network-online.target dbus.service containerd.service
Wants=network-online.target
Requires=containerd.service

[Service]
Type=simple
EnvironmentFile=-/etc/default/wendy-agent
WorkingDirectory=/var/lib/wendy-agent
ExecStart=/usr/local/bin/wendy-agent
Restart=always
RestartSec=2
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

# Write the environment defaults
sudo mkdir -p /etc/default
sudo tee /etc/default/wendy-agent <<'EOF'
WENDY_SYSTEMD_SERVICE_NAME=wendy-agent
# WENDY_NETWORK_MANAGER=auto
EOF

# Placeholder config
printf "{}\n" | sudo tee /etc/wendy-agent/config.json > /dev/null
sudo chmod 600 /etc/wendy-agent/config.json

sudo systemctl daemon-reload
sudo systemctl enable --now wendy-agent
```

---

## Offline dev container registry

The embedded OCI registry (`localhost:5000`) backed by containerd is built into `wendy-agent` and requires no separate installation. The registry allows the CLI to push container images directly to the device without going through an external registry.

For fully offline or WendyOS-pre-provisioned devices, the `wendyos-dev-registry-import.service` unit imports a pre-bundled OCI tar archive into containerd on first boot.

### Pre-bundled registry image

The archive is stored at:

```
/usr/share/wendyos/offline-images/containerd-registry.tar
```

The import service (`wendyos-dev-registry-import.service`) runs once and guards against re-running with a sentinel file:

```ini
ConditionPathExists=!/var/lib/wendyos/dev-registry-imported
ConditionPathExists=/usr/share/wendyos/offline-images/containerd-registry.tar
```

After a successful import the file `/var/lib/wendyos/dev-registry-imported` is created. Delete this file to force re-import on the next boot.

### Manual import (without the service)

```sh
sudo ctr -n default images import /usr/share/wendyos/offline-images/containerd-registry.tar
sudo ctr -n default images tag ghcr.io/wendylabsinc/containerd-registry:1.2.0 wendyos/containerd-registry:v1.2.0
sudo ctr -n default images label wendyos/containerd-registry:v1.2.0 containerd.io/gc.root=true
```

---

## Pre-pulling application images

For air-gapped deployments, OCI images must be pre-imported into containerd before the device loses network access. The agent's `WendyContainerService` accepts layers streamed from the CLI via `WriteLayer` + `CreateContainer`, so the CLI can upload images that were built and exported on a connected machine.

### Workflow

**On a connected machine:**

```sh
# Build and export the image as a tarball
docker build -t myapp:latest .
docker save myapp:latest -o myapp.tar
```

**Transfer the tarball to the device:**

```sh
scp myapp.tar user@device:/tmp/
```

**Import into containerd on the device:**

```sh
sudo ctr -n default images import /tmp/myapp.tar
```

The image is now available locally and the CLI can deploy it without a network pull.

### Layer deduplication

When the CLI runs `wendy run` it streams OCI layers to the device using the `WriteLayer` → `CreateContainer` flow. The CLI first calls `ListLayers` to see which layer digests containerd already has, and skips uploading layers that are already present. Pre-importing images via `ctr images import` means only missing or changed layers need to be transferred on subsequent deployments.

---

## Air-gapped agent updates

When the device has no internet access, use `wendy device update --binary` to provide a locally downloaded binary:

```sh
# On a connected machine: download the binary for the device architecture
curl -fsSL \
  "https://github.com/wendylabsinc/wendy-agent/releases/download/v0.5.0/wendy-agent-linux-arm64-0.5.0.tar.gz" \
  | tar -xz --strip-components=1 -C /tmp wendy-agent-linux-arm64/wendy-agent
mv /tmp/wendy-agent /tmp/wendy-agent-linux-arm64

# Push to the device
wendy --device <device-ip> device update --binary /tmp/wendy-agent-linux-arm64
```

The CLI computes the SHA-256 of the binary and sends it with the update command; the agent verifies it before replacing itself.

---

## Persistent volumes and offline storage

Persistent volumes are stored on the device at `/var/lib/wendy/volumes/<name>/`. They survive container restarts and image updates. For pre-seeding data (model weights, databases, etc.) in an air-gapped deployment, copy files directly to the volume path before starting the container:

```sh
sudo mkdir -p /var/lib/wendy/volumes/huggingface-cache
sudo cp -r /mnt/transfer/hf-models/* /var/lib/wendy/volumes/huggingface-cache/
```

The agent mounts this directory into the container when the `persist` entitlement references that volume name.
