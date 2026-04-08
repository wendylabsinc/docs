The Wendy-Agent, on *OCI enabled devices*, offers a container registry, which [clients](../../clients) can use to **upload** apps.

> **TODO**: The registry is currently a normal one, which means apps can also be deleted. This should not be allowed.

The standard container registry spec is available over plaintext HTTP for unprovisioned devices. 

> **TODO (urgent)**: Provisioned devices may only be accessed through HTTPS, using your [developer certificate](../../pki/) for authentication.

HTTPS is provided by the container registry when a device is provisioned.

> **TODO (urgent)**: The Container Registry must validate an authorized developer is accessing its contents.

The registry effectively acts as an HTTP(S) proxy for containerd, so when pushing a container to the registry, it is directly injected into `containerd`. This way we don't duplicate data between registry and containerd, saving disk space.

> **IDEA**: Containerd can also pull artifacts, for example, from [Wendy Cloud](../../cloud). This means we could disable the registry on production devices entirely.s