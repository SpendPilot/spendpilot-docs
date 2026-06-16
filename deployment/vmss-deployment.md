# VMSS deployment

Use the assets in `infra/vmss/`.

High-level steps:

1. Publish images to ACR.
2. Configure cloud-init and custom script bootstrap.
3. Supply environment variables through the VMSS model.
4. Place instances behind Azure Load Balancer or Application Gateway.
5. Keep PostgreSQL external and shared.

Important reminder:

- VMSS instances must stay stateless enough to recycle safely
