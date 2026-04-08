When an entitlement is set, the agent will apply the relevant changes to its OCI spec. For example, the `network` entitlement allows your app (container) to access IP networking.

Any entitlements _not_ set will reuslt in that resource _not_ being available.

Entitlements are [code signed](codesigning.md) as well, ensuring no privilege escalation.