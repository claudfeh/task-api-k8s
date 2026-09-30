# task-api on Kubernetes

Deploys the [task-api](https://github.com/claudfeh/task-api) app (Flask + Gunicorn + PostgreSQL 16) to Kubernetes — two ways: raw manifests and a Helm chart.

This is project 4 of a DevOps portfolio. Sibling projects:
- [task-api](https://github.com/claudfeh/task-api) — containerized Flask/PostgreSQL/nginx app
- CI/CD — GitHub Actions pipeline (test → build → push to GHCR) in the task-api repo
- [task-api-infra](https://github.com/claudfeh/task-api-infra) — Terraform infrastructure on AWS

## Architecture

```
                    ┌──────────────────────────────────────────────┐
                    │                  minikube                     │
  task-api.local →  │  Ingress (NGINX)                             │
                    │       │                                      │
                    │       ▼                                      │
                    │  Service "task-api" ──► Deployment (2× Flask)│
                    │       │                    │                 │
                    │       │              DB_HOST=postgres        │
                    │       ▼                    │                 │
                    │  Service "postgres" ◄──────┘                 │
                    │       │                                      │
                    │       ▼                                      │
                    │  Postgres 16 + PersistentVolumeClaim (1Gi)   │
                    └──────────────────────────────────────────────┘
```

Supporting objects: a **Secret** (database password), a **ConfigMap** (init.sql schema + seed data, mounted into Postgres on first boot).

## Layout

```
task-api-k8s/
├── manifests/          # Raw Kubernetes YAML — apply directly
│   ├── deployment.yaml / service.yaml / ingress.yaml
│   └── postgres-*.yaml # secret, configmap, pvc, deployment, service
└── helm/
    └── task-api/       # The same stack as a Helm chart
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
```

## Prerequisites

- [minikube](https://minikube.sigs.k8s.io/docs/start/), `kubectl`, and [Helm](https://helm.sh/docs/intro/install/)

## Option A — raw manifests

```bash
minikube start --driver=docker
minikube addons enable ingress

kubectl apply -f manifests/

# Wait for pods, then reach the app:
kubectl port-forward svc/task-api 5000:5000
# http://localhost:5000/health  → {"service":"task-api","database":"up"}
# http://localhost:5000/api/tasks → the seed tasks as JSON
```

Through the Ingress (host-based routing, faking DNS with a header):

```bash
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80
curl -H "Host: task-api.local" http://localhost:8080/api/tasks
```

## Option B — Helm chart

```bash
helm install task-api ./helm/task-api

# Change config without touching YAML:
helm upgrade task-api ./helm/task-api --set replicaCount=3
kubectl get pods   # a third app pod appears

helm uninstall task-api   # remove everything the chart installed
```

## Configuration (helm/task-api/values.yaml)

| Key | Default | Meaning |
|---|---|---|
| `replicaCount` | `2` | App pod copies (self-healing) |
| `image.repository` / `image.tag` | `ghcr.io/claudfeh/task-api` / `latest` | Public GHCR image from the CI pipeline |
| `service.port` | `5000` | Gunicorn's port |
| `ingress.host` | `task-api.local` | Hostname the Ingress routes |
| `postgres.dbName` / `dbUser` | `tasks` / `taskuser` | Database + role |
| `postgres.dbPassword` | `taskpass` | Demo only — override per environment: `--set postgres.dbPassword=...` |
| `postgres.storage` | `1Gi` | Database disk size |

## What this demonstrates

- **Deployments & self-healing** — pods are disposable; the Deployment keeps the replica count (`kubectl delete pod <name>` → a replacement appears)
- **Services & cluster DNS** — stable names/IPs in front of ever-changing pods; the app finds its DB via `DB_HOST=postgres`
- **Secrets vs ConfigMaps** — sensitive values vs plain config files
- **PersistentVolumeClaims** — the database survives pod restarts (1 replica: two Postgres writers would corrupt one disk)
- **Ingress** — one front door routing by hostname, the nginx role from the Compose setup
- **Helm** — the same stack packaged once, configured per environment via `values.yaml`; `helm template` renders, `install`/`upgrade`/`uninstall` manage the release lifecycle
