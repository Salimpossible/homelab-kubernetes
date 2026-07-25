# Homelab Kubernetes Media Stack

Deploys a media stack on k3s via Kustomize. This file is the source of truth for agents — read it before making changes.

## Scope

- Workloads (`apps/`): Plex, Jellyfin, Radarr, Sonarr, Prowlarr, Sportarr, qBittorrent (behind a Gluetun VPN sidecar).
- Platform (`infrastructure/`): namespaces, ingress-nginx, cert-manager, storage.
- `configs/`: application config artifacts consumed by workloads.
- `kustomization.yaml` (root): assembles the whole stack — this is the actual source of truth for what's deployed, not directory listings. `infrastructure/kustomization.yaml` is dead/unused, ignore it.
- Not currently deployed, present on disk only: `apps/clamav/`, `apps/media-scanner/` — excluded from root kustomization. Don't assume either is running.

## Non-negotiable conventions

- Keep ingress-nginx as the single public reverse proxy entrypoint.
- Keep app Services in `media` as `ClusterIP` unless the user explicitly asks for direct exposure.
- Keep TLS termination at ingress-nginx.
- Keep the cert-manager-managed wildcard secret pattern (`homelab-wildcard-tls`) unless the user requests a different model.
- When changing architecture behavior, update this file in the same change.

## Traffic Flow

`client -> DNS -> ingress LB (80/443) -> ingress rule match -> app Service -> app pod`

Example: `https://radarr.home.arpa -> radarr Service:7878`

For non-HTTP protocols (e.g. BitTorrent peer traffic), ingress-nginx stream mappings expose specific TCP/UDP ports on the same LoadBalancer and forward to an internal `ClusterIP` Service.

## Required Ingress Shape

- `spec.ingressClassName: nginx`
- `spec.rules[].host`
- `spec.rules[].http.paths[].backend.service.name`
- `spec.rules[].http.paths[].backend.service.port.number` (Service port)
- `spec.tls[].hosts`
- `spec.tls[].secretName`
- Recommended: `nginx.ingress.kubernetes.io/ssl-redirect: "true"`

## TLS Pattern

- Issuer: `ClusterIssuer/homelab-ca-issuer`
- Certificate: `Certificate/homelab-wildcard` in `media`
- Secret: `homelab-wildcard-tls` in `media`
- App ingresses reference that secret.

## Storage Strategy

- `local-path` = node-local, `ReadWriteOnce` config/state volumes (default StorageClass).
- `smb-media` = shared SMB-backed volumes, `ReadWriteMany`, `ReclaimPolicy: Retain`.
- Stateful app configs use `local-path`: Plex (`plex-config-laptop`), Jellyfin, Radarr, Sonarr, Sportarr, qBittorrent, Prowlarr, Gluetun.
- Shared library/workspaces use PVCs on `smb-media`: `media` (library) and `downloads-smb` (download workspace).
- Downloads workflow: qBittorrent writes to `downloads-smb`; Radarr, Sonarr, and Sportarr mount the same workspace for import management; media is then promoted into library paths on `smb-media`.
- Backup focus: app config PVCs and SMB share data.
- Scheduling implication: app config `local-path` PVCs are node-coupled; avoid moving those pods across nodes without a migration plan.
- Known orphaned PVCs still bound in-cluster but unused by any current manifest: `plex-config` (smb-media, superseded by `plex-config-laptop`), `clamav-config`, legacy `downloads` (local-path). Don't build on these; ask before deleting.

## Networking Extras

- `overlays/qbittorrent-no-vpn/`: patches qBittorrent to skip the Gluetun sidecar, for VPN troubleshooting only. Not applied by default.

## DNS

- All app hosts must resolve to the ingress IP.
- Missing DNS = ingress healthy but host unreachable.

## Fast Troubleshooting

- Resolve failure: DNS/hosts.
- `404`: bad ingress host/path match.
- `308`: redirect working.
- TLS warning: trust or hostname mismatch.
- TLS handshake failure: bad/missing TLS secret/key.
- `502/503`: Service endpoints/port issue.
- `401`: app auth reached (network path OK).

## Validation

Run after networking changes and report the output:

```bash
kubectl get ingress -n media
kubectl get svc -n ingress-system
kubectl get certificate -n media
kubectl get endpoints -n media
```
