# Homelab Kubernetes Media Stack

Media stack on k3s, deployed declaratively via Kustomize.

For architecture, conventions, and troubleshooting: see `CLAUDE.md`.

## Workloads

Plex, Jellyfin, Radarr, Sonarr, Prowlarr, Sportarr, qBittorrent (behind a Gluetun VPN sidecar).

## Repo Layout

- `apps/`: media workloads (Deployment, Service, Ingress, PVC, config/secret).
- `infrastructure/`: shared platform layers (namespaces, ingress, cert-manager, storage).
- `configs/`: application config artifacts consumed by workloads.
- `overlays/`: situational patches (e.g. `qbittorrent-no-vpn` for VPN troubleshooting).
- `kustomization.yaml`: root entrypoint assembling the stack.

## Quick Commands

```bash
kubectl get nodes
kubectl get ingress -n media
kubectl get svc -n ingress-system
kubectl get certificate -n media
```

## Apply

```bash
kubectl apply -k .
```
