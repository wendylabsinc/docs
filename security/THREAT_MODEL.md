# WendyOS Threat Model

This document describes the security boundaries, trust assumptions, and known mitigations for the WendyOS platform.

---

## Trust boundaries

| Boundary | Description |
|----------|-------------|
| App ↔ Host | Apps run in OCI containers. Entitlements are the only mechanism through which an app may access host devices or persistent storage. |
| Config ↔ Runtime | `wendy.json` is code-signed. An app cannot escalate its own privileges at runtime. |
| CLI ↔ Agent | The CLI communicates with the agent over gRPC (mTLS when provisioned). |
| Agent ↔ Cloud | The agent authenticates to Wendy Cloud with a device certificate issued by pki-core. |

---

## Entitlement security

All entitlements are validated at parse time (`AppConfig.Validate()`) and again defensively inside the apply functions. The following issues have been addressed:

### WDY-1015 — I2C device path traversal

**Risk:** A crafted `device` value such as `../sda` or `../../etc/passwd` could cause `applyI2C` to bind-mount an arbitrary host device into the container.

**Mitigation:** `device` is validated to match the pattern `i2c-N` where `N` is one or more ASCII digits. Any other value is rejected at `Validate()` and silently skipped in `applyI2C`. See [OCI Entitlements — i2c](../wendy-agent/oci/entitlements.md#i2c).

### WDY-1016 — Persist mount destination dot-dot traversal

**Risk:** A `path` value such as `/data/../etc` would be accepted as absolute, then silently resolved by `filepath.Clean` to `/etc` as the container mount destination, potentially shadowing sensitive container paths.

**Mitigation:** `path` is checked for `..` components on the **raw** (pre-Clean) value, in addition to the existing absolute-path check. Paths containing `..` are rejected at `Validate()` and silently skipped in `applyPersist`. See [OCI Entitlements — persist](../wendy-agent/oci/entitlements.md#persist).

---

## Code signing

App configs (`wendy.json`) are code-signed before deployment. The agent verifies the signature before applying entitlements, preventing an unprivileged process from modifying entitlements at runtime. See [OCI Code Signing](../wendy-agent/oci/codesigning.md).

---

## PKI and mTLS

Device certificates are issued by [pki-core](../pki/README.md). Provisioned devices serve their gRPC API on port 50052 with mTLS, requiring a matching client certificate for all operations.

---

## Reporting vulnerabilities

Please report security issues to security@wendylabs.com.
