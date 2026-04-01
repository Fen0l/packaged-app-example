# Packaged App Example — Podinfo on NKP

Demonstrates project-level GitOps deployment on NKP using a Helm chart stored in Harbor OCI registry, deployed via Flux on a workload cluster.

## Architecture

```
GitHub Repo                    Harbor OCI Registry              Workload Cluster
┌─────────────────┐           ┌──────────────────┐           ┌──────────────────────┐
│ deploy/          │           │ nkp/podinfo:1.0.0│           │ Project NS (my-app)  │
│  ├─ helmrepo     │─clone──▶  │ (Helm chart)     │           │                      │
│  ├─ helmrelease  │  via      └────────┬─────────┘           │  Deployment (2 pods) │
│  ├─ harbor-creds │  GitOps            │ pull chart          │  Service (ClusterIP) │
│  └─ ingress      │  Source            │                     │  ConfigMap           │
│                  │           ┌────────▼─────────┐           │  Ingress (Traefik)   │
│ templates/       │           │ Flux on workload │──install─▶│                      │
│  ├─ deployment   │           │ HelmRelease      │           └──────────────────────┘
│  ├─ service      │           └──────────────────┘
│  └─ configmap    │
└─────────────────┘
```

**Two paths, one repo:**

| Path | Delivered via | Contains | When to update |
|------|--------------|----------|----------------|
| `deploy/` | Git clone (GitopsRepository) | Flux CRs + infra manifests (Ingress, NetworkPolicy) | Change infra resources → git push |
| `templates/` + `values.yaml` | Harbor OCI (`helm push`) | Helm chart (app Deployment, Service, ConfigMap) | Change app code/config → repackage + push to Harbor |

## Prerequisites

- NKP 2.17 with a workspace and project (e.g., workspace `demo`, project `my-app`)
- GitopsRepository created in the project (NKP UI → Project → GitOps Sources → Add)
- Harbor accessible from the workload cluster (`harbor.itcs.local`)
- Podinfo container image available (public `ghcr.io/stefanprodan/podinfo:6.7.1` or pushed to Harbor)

## Setup

### 1. Push the Helm chart to Harbor

```bash
# Package the chart
helm package .
# → podinfo-1.0.0.tgz

# Login to Harbor (workaround for missing libsecret on WSL)
DOCKER_CONFIG=$(mktemp -d)
echo '{"credsStore":"","auths":{"harbor.itcs.local":{"auth":"'$(echo -n admin:Harbor12345 | base64)'"}}}' > ${DOCKER_CONFIG}/config.json
export DOCKER_CONFIG

# Push chart as OCI artifact
helm push podinfo-1.0.0.tgz oci://harbor.itcs.local/nkp --plain-http
```

### 2. Create the GitOps Source in NKP UI

1. Navigate to **Project → GitOps Sources → Add GitOps Source**
2. Set:
   - **Name:** `demo-podinfo`
   - **Repository URL:** `https://github.com/<org>/packaged-app-example.git`
   - **Branch:** `main`
   - **Path:** `./deploy`
3. Save — Flux clones the repo, applies `deploy/` manifests to the workload cluster

### 3. Verify

```bash
# On the workload cluster — check Flux resources
kubectl get helmrepositories,helmreleases -n <project-ns>

# Check pods
kubectl get pods -n <project-ns> -l app.kubernetes.io/name=podinfo

# NKP UI: Project → Helm Releases tab
```

## Updating the App

### Change app configuration (Helm values)

Edit `values.yaml` or `templates/`, then repackage and push:

```bash
# Bump version in Chart.yaml
# Edit values.yaml or templates as needed

helm package .
helm push podinfo-<new-version>.tgz oci://harbor.itcs.local/nkp --plain-http

# Update deploy/helmrelease.yaml → spec.chart.spec.version
git add -A && git commit -m "bump podinfo to <new-version>" && git push
```

### Change infra resources (Ingress, NetworkPolicy)

Edit files in `deploy/`, then push:

```bash
git add -A && git commit -m "update ingress host" && git push
# GitopsRepository syncs automatically (3min interval)
```

## Repository Structure

```
packaged-app-example/
├── Chart.yaml                 # Helm chart metadata (name: podinfo, version: 1.0.0)
├── values.yaml                # Default Helm values
├── .helmignore                # Ignore .git, temp files
├── templates/                 # Helm templates (packaged into chart)
│   ├── _helpers.tpl           #   Labels, names
│   ├── deployment.yaml        #   Deployment with probes, resource limits
│   ├── service.yaml           #   ClusterIP Service (9898/http, 9999/grpc)
│   ├── configmap.yaml         #   UI color + message
│   ├── hpa.yaml               #   HPA (optional, gated)
│   └── ingress.yaml           #   Ingress (optional, gated)
├── deploy/                    # Plain K8s manifests (applied via GitopsRepository)
│   ├── kustomization.yaml     #   Kustomize resource list
│   ├── harbor-creds.yaml      #   Harbor OCI registry credentials
│   ├── helmrepository.yaml    #   Flux HelmRepository → oci://harbor.itcs.local/nkp
│   ├── helmrelease.yaml       #   Flux HelmRelease → podinfo chart
│   └── ingress.yaml           #   Traefik Ingress
└── README.md
```
