# Threat Model

This document describes the security boundaries enforced by `wendy-agent` when running user app containers, and the mitigations in place against common attack vectors.

## Container isolation

Every app container deployed through `wendy-agent` runs with a hardened security configuration. The following restrictions are applied by default — no `wendy.json` entitlement is required to activate them.

### Linux capabilities

`wendy-agent` drops all non-essential Linux capabilities from containers. In particular:

| Capability | Status | Rationale |
|------------|--------|-----------|
| `CAP_SYS_PTRACE` | **Dropped** | Prevents a container from tracing host or sibling processes |
| `CAP_SYS_CHROOT` | **Dropped** | Prevents `chroot(2)` escapes to the host filesystem |

Capabilities are dropped in addition to Docker's default capability set, so containers receive a strict subset of what a plain `docker run` would grant.

### Seccomp profile (WDY-1099)

`wendy-agent` applies a custom seccomp (secure computing) filter to every container at runtime. The profile denies syscalls that are not needed for normal application workloads but are commonly abused in container-escape chains.

Key syscalls blocked by the profile:

| Syscall | Reason blocked |
|---------|---------------|
| `ptrace` | Process tracing / debugging of host and peer processes |
| `unshare` | Namespace creation — first step of unprivileged privilege escalation via `CLONE_NEWUSER` |

When a blocked syscall is invoked the kernel returns `EPERM` (errno 1) or `EACCES` (errno 13). The seccomp filter is enforced in addition to capability drops, providing defence-in-depth.

#### Verification

The CI integration tests `python-no-ptrace` and `python-no-unshare` automatically verify that these syscalls are blocked in every build. See [Integration Tests](../operations/testing/README.md#seccomp-tests-wdy-1099) for details.

## Entitlements and code signing

Access to hardware and system capabilities beyond the defaults must be explicitly declared in [`wendy.json`](../apps/wendy.json.md) as [entitlements](../apps/wendy.json.md#entitlements). Entitlements are code-signed, preventing a container from escalating its own privileges at runtime.

See [OCI Entitlements](../wendy-agent/oci/entitlements.md) and [Code Signing](../wendy-agent/oci/codesigning.md) for the full signing and verification flow.

## Network isolation

By default, containers have no network access. The `network` entitlement must be declared to enable outbound IP connectivity. The `network: host` mode shares the host network stack and requires explicit opt-in.

The OTEL telemetry receivers (ports 4317 and 4318) are bound to loopback only and are not reachable from the external network.

## Attack surface summary

| Attack vector | Mitigation |
|---------------|-----------|
| Process tracing of host/peers | `CAP_SYS_PTRACE` dropped; `ptrace` syscall blocked by seccomp |
| `chroot` filesystem escape | `CAP_SYS_CHROOT` dropped |
| Unprivileged namespace escape via `CLONE_NEWUSER` | `unshare` syscall blocked by seccomp |
| Unauthorised network access | Default-deny network; explicit `network` entitlement required |
| Privilege escalation via entitlement manipulation | Entitlements are code-signed |
