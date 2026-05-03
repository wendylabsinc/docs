# Threat Model

This document describes the security controls applied to containers running on WendyOS and tracks mitigations for known threat vectors.

## Container capability hardening

Every container receives a minimal Linux capability set through `defaultCapabilities()`. Capabilities are **not** granted unless there is an explicit, justified need.

### Default capability set

The following capabilities are granted to all containers:

| Capability | Reason |
|---|---|
| `CAP_CHOWN` | Change file ownership inside the container |
| `CAP_DAC_OVERRIDE` | Allow container processes to bypass file permission checks |
| `CAP_FSETID` | Set file set-user-ID and set-group-ID bits |
| `CAP_FOWNER` | Bypass permission checks for operations requiring file ownership |
| `CAP_MKNOD` | Create special files |
| `CAP_NET_RAW` | Use raw sockets |
| `CAP_SETGID` | Manipulate group IDs |
| `CAP_SETUID` | Manipulate user IDs |
| `CAP_SETFCAP` | Set file capabilities |
| `CAP_SETPCAP` | Transfer and drop capabilities |
| `CAP_NET_BIND_SERVICE` | Bind to low-numbered ports |
| `CAP_KILL` | Send signals to processes |
| `CAP_AUDIT_WRITE` | Write to the kernel audit log |

### Capabilities intentionally absent

| Capability | Rationale |
|---|---|
| `CAP_SYS_CHROOT` | **Removed (WDY-1099).** Containerised workloads already have their own mount namespace; `chroot(2)` inside a container provides no benefit and was a potential pivot point for a filesystem escape. |
| `CAP_SYS_PTRACE` | **Removed (WDY-1099).** Audio, camera, and video device access does not require the ability to trace arbitrary processes. This capability was the primary vector for a container escape via device entitlements. |

`CAP_NET_ADMIN` is **not** part of the default set. It is granted only when an explicit `host` network entitlement is applied, where it is required to manipulate the host network stack.

---

## Device entitlements (`SetDeviceCapabilities`)

When a container is granted a hardware device entitlement (camera, microphone, speakers, video), `SetDeviceCapabilities()` is called to add the minimal capabilities required for device access. As of WDY-1099:

- `CAP_SYS_PTRACE` is **not** added — confirmed unnecessary for any device access path.
- `CAP_SYS_CHROOT` is **not** added — the container mount namespace already isolates the filesystem.

---

## Seccomp profile

Every container built through `DefaultSpec()` receives a default seccomp filter (`defaultSeccomp()`). The profile uses an **allow-all default action** (`SCMP_ACT_ALLOW`) with targeted deny rules for known escape primitives:

| Syscall | Rule | Errno | Rationale |
|---|---|---|---|
| `ptrace` | Unconditional deny | `EPERM` | Process tracing enables reading/writing arbitrary process memory; no legitimate in-container use case. |
| `unshare` | Unconditional deny | `EPERM` | Namespace detachment can be combined with mount namespaces to escape container isolation. |
| `clone` | Deny when `CLONE_NEWUSER` (bit `0x10000000`) is set | `EPERM` | Unprivileged user-namespace creation is the classic container-escape primitive; blocked via `SCMP_CMP_MASKED_EQ` argument filter. |

The `clone`/`CLONE_NEWUSER` rule uses a masked-equal (`SCMP_CMP_MASKED_EQ`) argument check on argument index 0 so that legitimate `clone` calls (thread creation, `fork`-equivalent) are unaffected.

> **Note:** `clone3` is not present in the current deny list. If the kernel exposes `clone3` and user-namespace creation via `clone3` becomes a concern, the rule should be extended to cover it.

---

## Tracked findings

| ID | Severity | Status | Description |
|---|---|---|---|
| WDY-1099 | High | **Mitigated** | [TM-I-05] Broad container capabilities enabling host filesystem escape. Resolved by removing `CAP_SYS_PTRACE` and `CAP_SYS_CHROOT` from all capability sets and adding the default seccomp profile. |
