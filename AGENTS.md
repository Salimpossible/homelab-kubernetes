# Agent Bootstrap

Read these files first, in order:

1. `ARCHITECTURE.md`
2. `README.md`

## Non-negotiable conventions

- Keep ingress-nginx as the single public reverse proxy entrypoint.
- Keep app Services in `media` as `ClusterIP` unless user explicitly asks for direct exposure.
- Keep TLS termination at ingress-nginx.
- Keep cert-manager-managed wildcard secret pattern (`homelab-wildcard-tls`) unless user requests a different model.
- When changing architecture behavior, update `ARCHITECTURE.md` in the same change.

## Validation after networking changes

Run and report:

```bash
kubectl get ingress -n media
kubectl get svc -n ingress-system
kubectl get certificate -n media
kubectl get endpoints -n media
```