# Wendy for Mac

Wendy for Mac is in progress for teams that want the Wendy deploy, manage, and debug workflow on headless Apple Silicon Macs.

Unlike WendyOS devices, Mac targets are not flashed with a WendyOS image. They use a macOS-specific `wendy-agent` path on the existing operating system.

## Target Hardware

- Mac mini
- Mac Studio
- Other headless Apple Silicon Macs

## Planned Workflow

```bash
wendy discover
wendy device set-default
wendy run
```

The goal is to make a headless Mac behave like any other Wendy device from the developer machine: discover it, select it, deploy apps to it, and debug remotely.

## Status

Wendy for Mac is in progress. Cloud fleet management and Jetson AGX Orin support are both imminent, while Jetson AGX Thor support remains in progress.

## Build Configuration

### Swift 6 and Strict Concurrency

`WendyAgentCore` requires **Swift 6 language mode** (`swiftLanguageModes: [.v6]`). All targets in the package enable the following upcoming features to match the macOS UI app's concurrency defaults:

- `InferIsolatedConformances`
- `NonisolatedNonsendingByDefault`

These settings are applied uniformly across `WendyAgentCore`, `WendyAgentGRPC`, `WendyCloudGRPC`, `OpenTelemetryGRPC`, and the test target.

Shared Xcode build settings (Swift version, concurrency feature flags) are defined in `swift/WendyAgentMac/Configuration/Default.xcconfig` and apply to all targets. Platform SDK selection remains target-specific.

### Main-Actor Default Isolation

The `WendyAgentMac` UI app target sets `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`. This means unannotated code in the macOS UI shell defaults to the main actor. Non-UI work belongs in `WendyAgentCore` or must declare its own isolation explicitly.
