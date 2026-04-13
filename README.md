# Homelab Kubernetes Media Stack

This repository deploys a media stack on Kubernetes using Kustomize.

Use these two docs only:

- `README.md` (this file): quick entrypoint.
- `ARCHITECTURE.md`: precise architecture, traffic flow, TLS/cert-manager, and operational patterns.

## Scope

- Workloads: Plex, qBittorrent (behind VPN sidecar pattern), Radarr, Sonarr, ClamAV.
- Platform: ingress-nginx, cert-manager, storage and namespace infrastructure.
- Deployment style: declarative manifests via kustomize.

## Repo Overview

- `apps/`: media workloads (Deployment, Service, Ingress, PVC, app config/secret).
- `infrastructure/`: shared platform layers (namespaces, ingress, cert-manager, storage, monitoring, network policies).
- `configs/`: application config artifacts consumed by workloads.
- `docker/`: legacy compose files (not the primary deployment path).
- `kustomization.yaml`: root entrypoint assembling the stack.
- `ARCHITECTURE.md`: source-of-truth for networking, TLS, and storage patterns.

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

## Read First

For implementation and troubleshooting details, read `ARCHITECTURE.md`.
