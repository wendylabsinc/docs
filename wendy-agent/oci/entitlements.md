# OCI Entitlements

The wendy-agent OCI runtime applies entitlements declared in `wendy.json` to the container's OCI spec before launch. This page documents how each entitlement is validated and applied, including the security rules enforced at both parse time and apply time.

---

## Validation

All entitlements are validated by `AppConfig.Validate()` before any container is started. Validation errors prevent the app from launching and surface a descriptive error message.

### `persist` validation

| Rule | Error |
|------|-------|
| `path` must be non-empty | `persist entitlement requires a path` |
| `path` must be absolute (start with `/`) | `persist path must be absolute` |
| `path` must not contain `..` components | `persist path must not contain '..' components` |

**Why:** Accepting a relative path or a path containing `..` components (e.g. `/data/../etc`) could silently resolve to a sensitive host directory when `filepath.Clean` is applied. Validation rejects these paths before they reach the apply function.

### `i2c` validation

| Rule | Error |
|------|-------|
| `device` must be non-empty | `i2c entitlement requires a device` |
| `device` must match `i2c-N` (digits only after the hyphen) | `i2c device must be in i2c-N format` |

**Why:** Constructing a `/dev/` path from an arbitrary `device` string (e.g. `../sda`, `../../etc/passwd`) could grant access to unintended host devices. Only `i2c-<digits>` names are accepted.

---

## Apply-time security

In addition to `Validate()`, the apply functions enforce the same rules defensively (belt-and-suspenders). Invalid entitlements are silently skipped at apply time so that a partially corrupted config cannot mount unexpected paths.

### `applyPersist`

1. Checks that `ent.Path` is absolute.
2. Splits `ent.Path` on `/` and rejects any component equal to `..`.
3. Only then calls `filepath.Clean(ent.Path)` to produce the canonical destination.

This means `/data/../etc` is **rejected** outright rather than silently resolved to `/etc`.

The host backing directory is always rooted at `/var/lib/wendy/volumes/<name>` and created with `0755` permissions before the bind mount is added to the spec.

### `applyI2C`

1. Checks that `ent.Device` has the prefix `i2c-`.
2. Checks that the suffix after `i2c-` is non-empty and contains only ASCII digits (`0`–`9`).
3. Constructs the device path as `filepath.Clean("/dev/" + ent.Device)`.

This means device names like `../sda`, `i2c-1/../sda`, `i2c-1a`, or bare `sda` are **rejected** and produce no mount entry in the spec.

---

## Entitlement reference

### `persist`

Mounts a persistent volume from the host into the container. The volume survives container restarts and re-deployments.

```json
{
  "type": "persist",
  "name": "my-app",
  "path": "/data"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `name` | yes | Volume namespace. Apps sharing the same name share storage. |
| `path` | yes | **Absolute** path inside the container with no `..` components. |

The host directory `/var/lib/wendy/volumes/<name>` is created automatically. The mount is a `rbind` bind mount with `nosuid` and `noexec` options.

**Valid examples:**

```json
{ "type": "persist", "name": "db", "path": "/var/lib/mydb" }
{ "type": "persist", "name": "models", "path": "/models" }
```

**Rejected examples:**

```json
{ "type": "persist", "name": "x", "path": "relative/path" }
{ "type": "persist", "name": "x", "path": "../escape" }
{ "type": "persist", "name": "x", "path": "/data/../etc" }
```

---

### `i2c`

Grants access to a specific I2C bus by bind-mounting its device node into the container.

```json
{
  "type": "i2c",
  "device": "i2c-1"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `device` | yes | I2C device name in `i2c-N` format (digits only after the hyphen). |

The device node `/dev/<device>` is bind-mounted into the container with the same path.

**Valid examples:**

```json
{ "type": "i2c", "device": "i2c-0" }
{ "type": "i2c", "device": "i2c-1" }
{ "type": "i2c", "device": "i2c-10" }
```

**Rejected examples:**

```json
{ "type": "i2c", "device": "../sda" }
{ "type": "i2c", "device": "i2c-1a" }
{ "type": "i2c", "device": "i2c-" }
{ "type": "i2c", "device": "sda" }
```

---

## Security notes

Both WDY-1015 (`i2c` path traversal) and WDY-1016 (`persist` dot-dot destination) are mitigated by a two-layer approach:

1. **Parse-time gate (`Validate()`)** — the primary enforcement point. Invalid configs are rejected before any container lifecycle code is reached.
2. **Apply-time guard (`applyI2C` / `applyPersist`)** — a defensive secondary check that ensures no mount is added even if validation is bypassed or the config is mutated in memory.

The test suite (in `go/internal/agent/oci/entitlements_test.go`) covers:

- `TestApplyI2C_PathTraversal` — traversal device names produce no mounts.
- `TestApplyI2C_ValidDevice` — `i2c-1` is still mounted correctly.
- `TestApplyPersist_PathTraversalDestination` — relative and traversal paths are rejected.
- `TestApplyPersist_DotDotInDestination` — `/data/../etc` is not resolved to `/etc` and mounted.
