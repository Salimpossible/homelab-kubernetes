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
  - Exception: `apps/jellyfin/service.yaml` is `NodePort` (`nodePort: 30096`), by explicit user request, so mobile clients (e.g. iOS app) on the LAN can reach it directly at `http://<node-ip>:30096` without needing `home.arpa` DNS resolution — mobile OSes have no hosts-file/local-DNS override, so ingress's Host-header-based routing (`jellyfin.home.arpa`) can't be reached by bare IP alone (confirmed: bare-IP request to ingress returns 404, only a matching `Host` header routes correctly). Plain HTTP, no TLS, on this path.
    - `apps/jellyfin/deployment.yaml` must NOT set `JELLYFIN_PublishedServerUrl` (removed 2026-08). That env var forces Jellyfin to advertise one fixed address to every client regardless of how they connected; it was set to `https://jellyfin.home.arpa`, which broke the NodePort path — LAN mobile clients would connect fine over `http://<node-ip>:30096` but then get redirected to the `home.arpa` HTTPS URL for follow-up API/session calls, which mobile devices can't resolve/trust, surfacing as a generic "could not find the server" error even though the TCP path was healthy. Without the override, Jellyfin infers the advertised address per-request, so both the ingress HTTPS path and the NodePort HTTP path keep working simultaneously.
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
- `smb-media` = shared SMB-backed volumes, `ReadWriteMany`, `ReclaimPolicy: Retain`, provisioner `smb.csi.k8s.io`, no `subDir` parameter set — each dynamically-provisioned PVC gets its own opaque top-level folder under the share root (`//192.168.1.21/Public`). Because of that, this StorageClass is deliberately backed by a **single** PVC (see below) rather than one PVC per logical volume — two PVCs on `smb-media` are two unrelated backend folders even though they share a physical server, which breaks same-filesystem renames between them.
- Stateful app configs use `local-path`: Plex (`plex-config-laptop`), Jellyfin, Radarr, Sonarr, Sportarr, qBittorrent, Prowlarr, Gluetun.
- Shared library and downloads workspace: a single PVC, `media`, on `smb-media`. All apps mount it via `subPath`, never full-root except `media-scanner` (undeployed), which would need to see the whole tree in one pod if ever re-enabled. Canonical top-level layout inside the PVC: `downloads/`, `movies/`, `tv/`, `sports/`.
  - qBittorrent, Radarr, Sonarr, Sportarr mount `subPath: downloads` at `/downloads`.
  - Radarr/Sonarr/Sportarr additionally mount their own genre folder (`subPath: movies|tv|sports`) at `/movies`, `/tv`, `/sports`.
  - Plex and Jellyfin mount three separate subPaths (`movies`, `tv`, `sports`) at `/media/movies`, `/media/tv`, `/media/sports` — never `downloads`, so neither can ever index an in-progress download.
- Downloads workflow: qBittorrent writes into `downloads/`; Radarr/Sonarr/Sportarr each mount the same `downloads/` subPath plus their own genre subPath and handle their own import (their built-in Completed Download Handling moves/renames files from `downloads/` into `movies/`/`tv/`/`sports/`). There is currently no antivirus/scan step in this pipeline — qBittorrent previously had a post-download hook (`qbit-on-complete.sh`) that spawned a Job to clamscan and promote each file via a `cp`+`rm`, but its trigger (`[AutoRun]` in qBittorrent's own config) was `enabled=false` — dead code, never actually invoked — so the hook, its ConfigMap, and its `qbittorrent-job-trigger` ServiceAccount/Role/RoleBinding were removed entirely (2026-08). If scanning is wanted again later, `apps/clamav/` and `apps/media-scanner/` still exist on disk (excluded from root `kustomization.yaml`) as a starting point, but note both were written assuming the old two-PVC layout and would need updating for the single-PVC `subPath` model above.
- Backup focus: app config PVCs and SMB share data.
- Scheduling implication: app config `local-path` PVCs are node-coupled; avoid moving those pods across nodes without a migration plan.
- Known orphaned PVCs still bound in-cluster but unused by any current manifest: `plex-config` (smb-media, superseded by `plex-config-laptop`), `clamav-config`, legacy `downloads` (local-path), and — once the single-PVC migration below is complete and verified — `downloads-smb` (smb-media, superseded by subPaths on `media`). Don't build on these; ask before deleting.
- Migration note: `downloads-smb` was retired in favor of a `downloads/` subPath on the `media` PVC (2026-08) specifically to fix the copy-not-rename issue above and to standardize subPath scoping across apps. If you still see a live `downloads-smb` PVC/PV, its backend data needs migrating into `media`'s `downloads/` folder before it's safe to delete — see git history for the migration runbook.

## Secrets

- `infrastructure/storage/secret-env.yaml` (the `media-stack-env` Secret, consumed by qBittorrent/Gluetun and Plex) is a plain Kubernetes Secret manifest referenced by the root `kustomization.yaml`, but the file itself is **gitignored** (`**/secret-env.yaml` in `.gitignore`) and has never been committed — it only exists locally on whichever machine applies the stack. That avoids a git-history leak, but also means it's un-versioned, unbacked-up, and single-machine — nothing enforces that it matches the live cluster.
- Keep this file in sync with whatever is actually live in the cluster (`kubectl get secret media-stack-env -n media -o yaml`) before running `kubectl apply -k .`. It has drifted before (missing `WIREGUARD_ADDRESSES`/`WIREGUARD_ENDPOINT_IP`/`WIREGUARD_ENDPOINT_PORT` keys entirely, stale `WIREGUARD_PUBLIC_KEY`) — because it's local-only, a stale copy on the applying machine will silently revert a live fix with no diff/review to catch it. `.env.example` at the repo root is the template for its shape, not a live source of truth.

## Networking Extras

- `overlays/qbittorrent-no-vpn/`: patches qBittorrent to skip the Gluetun sidecar, for VPN troubleshooting only. Not applied by default.
- Gluetun connects to a self-hosted "custom" WireGuard VPN server (not a commercial VPN provider), currently an AWS EC2 instance. `WIREGUARD_ENDPOINT_IP`/`WIREGUARD_PUBLIC_KEY` in `media-stack-env` must match that server's actual current public IP and WireGuard interface public key — if the server's keypair or IP ever changes and this secret isn't updated to match, the tunnel fails silently (handshake drops with no error, looks like a generic network/DNS problem). Cross-check both sides with `wg show` on the server before assuming a k8s-side networking bug.

## Image Versioning

- qBittorrent and Gluetun (`apps/qbittorrent/deployment.yaml`) intentionally use major-version-locked floating tags (`linuxserver/qbittorrent:5`, `qmcgaw/gluetun:v3`) with `imagePullPolicy: Always`, not digest pins — the user prefers automatic patch/minor security updates over strict reproducibility for these two images. This only takes effect on pod (re)creation (rollout, reschedule, crash-restart), not live in a running container. Don't "fix" this back to digest pinning without asking; it's a deliberate choice, not an oversight.

## Monitoring

- `infrastructure/monitoring/homepage/` (gethomepage/homepage, namespace `monitoring`) is the only monitoring in this stack — no Prometheus/Grafana/Alertmanager exist. It's a passive dashboard someone has to look at, not a pushed/paged alert.
- Every app already gets a live status tile for free via Ingress annotations (`gethomepage.dev/enabled: "true"`, `gethomepage.dev/group`, etc. — see any `apps/*/ingress.yaml`), which Homepage auto-discovers (`kubernetes.yaml` sets `mode: cluster`, `ingress: true`). **Don't add manual per-app entries to `services.yaml`** (in `homepage-configmap.yaml`) for apps that already have an ingress — that just duplicates the tile. `services.yaml` should stay limited to things with no ingress of their own (currently just the Homepage self-link).
- **Known limitation, not yet fixed:** Homepage's Kubernetes status check (`src/pages/api/kubernetes/status/[...service].js` upstream) only reads pod **phase** (`Running`/`Succeeded` vs anything else) — it does not check container readiness (`containerStatuses[].ready`). A pod stuck failing its `readinessProbe` still reports phase `Running`, so the dashboard tile stays green. This means gluetun's `readinessProbe`/`livenessProbe` (see Image Versioning/Networking Extras — hits its own health server at `HEALTH_SERVER_ADDRESS: ":9999"`, path `/`) makes `kubectl get pods` show `1/2 Ready` correctly, but does **not** flip the qBittorrent tile on Homepage — the exact silent-tunnel-failure scenario from the original incident still shows "RUNNING" on the dashboard today.
- **Next step to actually fix this** (not yet done): add a `customapi` widget on the qBittorrent tile pointed directly at gluetun's health endpoint (already exposed on port 9999 inside the pod) instead of relying on the generic Kubernetes status widget. Needs: (1) a small `ClusterIP` Service in `media` exposing the `vpn-torrent` pod's port 9999, since Homepage's container can't reach `127.0.0.1:9999` inside another pod's netns directly; (2) a `customapi` (or similar) widget block in `homepage-configmap.yaml`'s `widgets.yaml`/`services.yaml` reading that endpoint. This is the only way to get true tunnel-health visibility on the dashboard — pod-phase status can't do it.

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
