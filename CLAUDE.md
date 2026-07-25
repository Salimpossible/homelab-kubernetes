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

## Secrets

- `infrastructure/storage/secret-env.yaml` (the `media-stack-env` Secret, consumed by qBittorrent/Gluetun and Plex) is a plain Kubernetes Secret manifest referenced by the root `kustomization.yaml`, but the file itself is **gitignored** (`**/secret-env.yaml` in `.gitignore`) and has never been committed — it only exists locally on whichever machine applies the stack. That avoids a git-history leak, but also means it's un-versioned, unbacked-up, and single-machine — nothing enforces that it matches the live cluster.
- Keep this file in sync with whatever is actually live in the cluster (`kubectl get secret media-stack-env -n media -o yaml`) before running `kubectl apply -k .`. It has drifted before (missing `WIREGUARD_ADDRESSES`/`WIREGUARD_ENDPOINT_IP`/`WIREGUARD_ENDPOINT_PORT` keys entirely, stale `WIREGUARD_PUBLIC_KEY`) — because it's local-only, a stale copy on the applying machine will silently revert a live fix with no diff/review to catch it. `.env.example` at the repo root is the template for its shape, not a live source of truth.

## Networking Extras

- `overlays/qbittorrent-no-vpn/`: patches qBittorrent to skip the Gluetun sidecar, for VPN troubleshooting only. Not applied by default.
- Gluetun connects to a self-hosted "custom" WireGuard VPN server (not a commercial VPN provider), currently an AWS EC2 instance. `WIREGUARD_ENDPOINT_IP`/`WIREGUARD_PUBLIC_KEY` in `media-stack-env` must match that server's actual current public IP and WireGuard interface public key — if the server's keypair or IP ever changes and this secret isn't updated to match, the tunnel fails silently (handshake drops with no error, looks like a generic network/DNS problem). Cross-check both sides with `wg show` on the server before assuming a k8s-side networking bug.

## Image Versioning

- qBittorrent and Gluetun (`apps/qbittorrent/deployment.yaml`) intentionally use major-version-locked floating tags (`linuxserver/qbittorrent:5`, `qmcgaw/gluetun:v3`) with `imagePullPolicy: Always`, not digest pins — the user prefers automatic patch/minor security updates over strict reproducibility for these two images. This only takes effect on pod (re)creation (rollout, reschedule, crash-restart), not live in a running container. Don't "fix" this back to digest pinning without asking; it's a deliberate choice, not an oversight.

## Monitoring

- `infrastructure/monitoring/homepage/` (gethomepage/homepage, namespace `monitoring`) is the only monitoring in this stack — no Prometheus/Grafana/Alertmanager exist. It's a passive dashboard, not a pushed/paged alert.
- Homepage's `ClusterRole` already grants cluster-wide `get`/`list` on `pods`/`namespaces`/`nodes`, and `kubernetes.yaml` sets `mode: cluster`, `labelSelector: app`. `services.yaml` (in `homepage-configmap.yaml`) lists every currently-deployed app with `namespace: media` + `app: <label value>`, which makes each tile show live pod-readiness status instead of being a static link.
- Adding a new app to the stack means also adding a matching `services.yaml` entry (namespace + `app` label value) or it silently won't show status on the dashboard, just nothing.
- Status is only as good as the container's probes. A `Running` pod with a probe-less container looks healthy on the dashboard even if that container is internally broken — this bit us with the Gluetun sidecar (see Networking Extras): it had no probe at all, so a dead WireGuard tunnel never showed as unhealthy anywhere. Gluetun's container now has a `readinessProbe`/`livenessProbe` against its own built-in health server (`HEALTH_SERVER_ADDRESS: ":9999"`, path `/`) so a tunnel failure flips the `vpn-torrent` pod to not-ready and the qBittorrent tile on Homepage goes red — not just when the pod crashes. If you add a probe-less sidecar to any other app, its failures similarly won't surface anywhere.

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
