When an entitlement is set, the agent applies the relevant changes to the container's OCI spec. For example, the `network` entitlement allows your app to access IP networking.

Any entitlement _not_ declared in `wendy.json` will result in that resource being unavailable to the app.

Entitlements are [code signed](codesigning.md), ensuring no privilege escalation.

For the full list of available entitlements and their options, see [wendy.json — Entitlements](../../apps/wendy.json.md#entitlements-1).