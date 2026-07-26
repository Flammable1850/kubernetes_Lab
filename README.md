# Media Stack — Kubernetes (k3s)

Docker-images van de mediaserver stack, gedeployed als losstaand k3s-cluster,
GitOps-beheerd via ArgoCD vanuit GitLab.

## Overview

| Item | Value |
|---|---|
| Distributie | k3s |
| Host | Losstaande Proxmox VM |
| Namespace | `media` |
| Ingress controller | Traefik (ingebouwd in k3s) |
| Storage provisioner | `local-path` |
| GitOps | ArgoCD, sync vanuit GitLab repo |

## Architecture

```
Proxmox Host
└── VM — k3s (single-node)
    └── namespace: media
        ├── jellyfin       (Deployment, HW transcoding via GPU device plugin)
        ├── sonarr         (TV automation)
        ├── radarr         (movie automation)
        ├── prowlarr       (indexer manager — syncs naar Sonarr/Radarr)
        ├── qbittorrent    (torrent client, netwerk via gluetun)
        ├── gluetun        (VPN sidecar — kill-switch voor qBittorrent)
        └── Traefik IngressRoute (entry point voor alle web UI's)
```

## Cluster Config

```yaml
# 00-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: media
```

**GPU device plugin:** vereist voor hardware transcoding in Jellyfin. In tegenstelling
tot `/dev/dri`-passthrough in een LXC-container is dit in k3s niet automatisch —
installeer de NVIDIA/Intel device plugin op de node en verifieer met:
```bash
kubectl get nodes -o json | jq '.items[].status.allocatable'
```
Confirm dat `nvidia.com/gpu` (of het Intel-equivalent) als allocatable resource
verschijnt voordat GPU-requests in een Deployment worden gezet.

## Kubernetes Manifests

### Resources per service

| Resource | Purpose | Notes |
|---|---|---|
| Namespace | `media` | Isoleert de hele stack |
| PVC — `jellyfin-config` | Config/DB/metadata cache | 5Gi, `local-path` |
| PVC — `jellyfin-media` | Media library | 500Gi, `local-path`; naar RWX/NFS als andere pods erin schrijven |
| Deployment — Jellyfin | Media server | `strategy: Recreate`, GPU device request, PUID/PGID/TZ env |
| Deployment — Sonarr/Radarr | TV/film automation | Praat met Prowlarr + qBittorrent |
| Deployment — Prowlarr | Indexer manager | Voedt Sonarr/Radarr |
| Deployment — qBittorrent + gluetun sidecar | Download client + VPN kill-switch | `NET_ADMIN` capability, `/dev/net/tun` mount |
| Service | ClusterIP per app | Interne routing |
| IngressRoute | Traefik | Externe toegang per host |

### Example Deployment skeleton

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jellyfin
  namespace: media
  labels:
    app: jellyfin
spec:
  replicas: 1
  strategy:
    type: Recreate  # voorkomt dat twee pods om dezelfde RWO-volumes strijden
  selector:
    matchLabels:
      app: jellyfin
  template:
    metadata:
      labels:
        app: jellyfin
    spec:
      containers:
        - name: jellyfin
          image: jellyfin/jellyfin:<tag>
          ports:
            - name: http
              containerPort: 8096
          env:
            - name: PUID
              value: "<uid>"
            - name: PGID
              value: "<gid>"
            - name: TZ
              value: "Europe/Amsterdam"
          resources:
            limits:
              <gpu-vendor>.com/gpu: 1
          volumeMounts:
            - name: config
              mountPath: /config
            - name: media
              mountPath: /media
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: jellyfin-config
        - name: media
          persistentVolumeClaim:
            claimName: jellyfin-media
```

> Placeholders (`<tag>`, `<uid>`, `<gid>`, `<gpu-vendor>`) invullen voor gebruik.
> Secrets (VPN-credentials, API keys) via Kubernetes `Secret` objects, niet in de
> manifests zelf — en uitgesloten van version control als losse waardebestanden.

## Storage

- `local-path` provisioner voor beide PVC's — voldoende zolang het cluster
  single-node blijft.
- Bij schrijftoegang vanuit meerdere pods naar dezelfde media-map: overstappen naar
  ReadWriteMany via een NFS-export vanaf Proxmox (bijv. met `democratic-csi`).

## Resource Limits

- CPU/memory requests en limits per Deployment instellen in de container-spec
  (`resources.requests` / `resources.limits`).
- GPU-limit expliciet zetten op de Jellyfin-Deployment zodra de device plugin actief is.

## Hardening

- RBAC per namespace, geen cluster-admin voor applicatie-service-accounts.
- Secrets als Kubernetes `Secret` objects, niet als plain env-values in manifests.
- `NET_ADMIN` alleen op de gluetun-container, niet breder toegekend.
- Images met vaste tags i.p.v. `latest`, consistent met de voorkeur voor handmatige
  image-updates i.p.v. auto-update tooling.
