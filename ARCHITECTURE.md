# Cluster Architecture

## Pattern

- One public reverse proxy: `ingress-nginx`.
- One public entrypoint: `infrastructure/ingress/service.yaml` as `LoadBalancer` on `80/443`.
- Media app Services stay internal: `ClusterIP`.
- Routing is in app `Ingress` resources.
- TLS terminates at ingress-nginx.
- Certs are managed by cert-manager.

## Traffic Flow

`client -> DNS -> ingress LB (80/443) -> ingress rule match -> app Service -> app pod`

Example: `https://radarr.home.arpa -> radarr Service:7878`

For non-HTTP protocols (for example BitTorrent peer traffic), ingress-nginx stream mappings can expose specific TCP/UDP ports on the same ingress LoadBalancer and forward them to an internal `ClusterIP` Service.

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

## Rules

- Keep `media` Services as `ClusterIP` unless direct exposure is explicitly requested.
- Do not add app `LoadBalancer`/`NodePort` by default.
- If architecture changes, update this file in same change.

## Storage Strategy

- `local-path` = node-local, `ReadWriteOnce` config/state volumes.
- `smb-media` = shared SMB-backed volumes, `ReadWriteMany`, `ReclaimPolicy: Retain`.
- Stateful app configs use `local-path`: Plex, Radarr, Sonarr, Sportarr, qBittorrent, Prowlarr, Gluetun, ClamAV.
- Shared library/workspaces use PVCs on `smb-media`: `media` (library) and `downloads-smb` (download workspace).
- Downloads workflow: qBittorrent → `smb-media` `downloads-smb` → ClamAV scan → promote into library paths on `smb-media`.
- Backup focus: app config PVCs and SMB share data.
- Scheduling implication: app config `local-path` PVCs are node-coupled; avoid moving those pods across nodes without migration plan.

## DNS

- All app hosts must resolve to ingress IP.
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

```bash
kubectl get ingress -n media
kubectl get svc -n ingress-system
kubectl get endpoints -n media
kubectl get certificate -n media
```