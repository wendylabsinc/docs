Authenticates the CLI with Wendy Cloud. Opens a browser to the cloud dashboard, waits for the OAuth callback, generates a key pair and CSR, then issues and stores an mTLS certificate. Subsequent commands that connect to provisioned devices use this certificate automatically.

Optionally accepts `--cloud` (dashboard URL) and `--cloud-grpc` (gRPC endpoint) to point at a non-default cloud instance.
