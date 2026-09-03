# cd-process-golang-api

GitOps repository that drives the continuous deployment of the [books-go-api](https://github.com/cordonezjolon/books-go-api) Golang API to Kubernetes via [Argo CD](https://argo-cd.readthedocs.io/).

## How it works

Argo CD watches this repository and syncs the manifests under `apps/` into the target Kubernetes cluster. When the application's CI pipeline builds a new image, it opens a commit here that bumps the image tag in [apps/books-api/deployment.yaml](apps/books-api/deployment.yaml) (see the `chore: bump cordonezjolon/books-go-api to develop-N` commits in the git history). Argo CD then picks up the change and rolls it out automatically.

```
CI (books-go-api repo)
   │  builds & pushes image cordonezjolon/books-go-api:develop-N
   ▼
this repo (image tag bump commit)
   │
   ▼
Argo CD (auto-sync, prune, self-heal)
   │
   ▼
Kubernetes cluster
```

## Repository structure

```
.
├── apps/
│   └── books-api/          # Raw Kubernetes manifests for the books-api application
│       ├── deployment.yaml # Namespace + Deployment (image tag is bumped here per release)
│       ├── service.yaml    # NodePort Service exposing the app on port 80 -> 8080
│       ├── ingress.yaml    # nginx Ingress + cert-manager TLS
│       └── secret.yaml     # DopplerSecret CR that syncs app secrets from Doppler
└── environments/
    ├── dev/
    │   └── books-api.yaml  # Argo CD Application pointing at apps/books-api (targetRevision: develop)
    └── stage/               # Reserved for a staging Argo CD Application (not yet configured)
```

- **apps/**: the actual workload manifests that get deployed into the cluster.
- **environments/**: Argo CD `Application` resources, one per environment, each pointing Argo CD at a path/branch in this repo.

## Components

| Resource | Purpose |
|---|---|
| `Namespace: books-api-development` | Isolates the application's resources. |
| `Deployment: books-api` | Runs 2 replicas of the API container, pulling env vars from the `books-go-api-secret` secret. |
| `Service: books-service` | NodePort service routing traffic to the pods on port 8080. |
| `Ingress: books-ingress` | Exposes the service externally via nginx, with TLS issued by `letsencrypt-stage` (cert-manager). |
| `DopplerSecret: books-go-api-doppler-secret` | Managed by the [Doppler Kubernetes Operator](https://docs.doppler.com/docs/kubernetes-operator); syncs secrets from the `books-go-api` Doppler project (`dev` config) into the `books-go-api-secret` Kubernetes Secret. |
| `Application: books-api` (Argo CD) | Points Argo CD at `apps/books-api` on the `develop` branch, with automated sync, pruning, and self-healing enabled. |

## Prerequisites

The target cluster must have the following already installed for these manifests to sync successfully:

- [Argo CD](https://argo-cd.readthedocs.io/)
- [ingress-nginx](https://kubernetes.github.io/ingress-nginx/)
- [cert-manager](https://cert-manager.io/) with a `letsencrypt-stage` `ClusterIssuer`
- [Doppler Kubernetes Operator](https://docs.doppler.com/docs/kubernetes-operator), configured with a `doppler-token-books-go-api` token secret in the `doppler-operator-system` namespace

## Adding a new environment

To add a staging (or other) environment:

1. Add manifests under `apps/` (or reuse `apps/books-api` if the same workload is deployed with different config per environment).
2. Add an Argo CD `Application` under `environments/<env>/` pointing at the desired path and `targetRevision`.
3. Apply the `Application` manifest to the cluster (or let Argo CD's own app-of-apps pattern pick it up, if configured).
