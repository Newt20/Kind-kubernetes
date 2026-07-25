# BMI Health Tracker - 3-Tier Kubernetes Deployment

A 3-tier web application (React frontend, Node.js/Express backend, PostgreSQL database) deployed on a local Kind (Kubernetes-in-Docker) cluster.

---

## Architecture

```
                    ┌─────────────────────┐
                    │   NGINX Ingress      │  :80 (kind hostPort)
                    │   Controller          │
                    └──────────┬───────────┘
                               │ /
                    ┌──────────▼───────────┐
                    │   frontend-svc        │  ClusterIP :80
                    │   (React + Nginx)      │  2 replicas
                    └──────────┬───────────┘
                               │ /api/* (nginx reverse proxy,
                               │ resolved via cluster DNS)
                    ┌──────────▼───────────┐
                    │   backend-svc         │  ClusterIP :3000
                    │   (Node.js + Express)  │  3 replicas
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │   postgres-svc        │  ClusterIP :5432
                    │   (PostgreSQL 16)      │  PVC-backed storage
                    └───────────────────────┘
```

Cluster topology: 1 control-plane node + 2 worker nodes (Kind). Backend and frontend pods are spread across both worker nodes via pod anti-affinity; Postgres is pinned to a specific worker node to match its PersistentVolume's hostPath.

## Project Structure

```
app/
├── backend/                  # Node.js + Express API
│   ├── src/                  # server.js, routes.js, db.js, calculations.js
│   ├── migrations/           # PostgreSQL schema migrations
│   ├── Dockerfile
│   └── .env.example
├── frontend/                  # React + Vite SPA
│   ├── src/
│   ├── Dockerfile
│   ├── nginx.conf            # SPA routing + /api/* reverse proxy to backend-svc
│   └── .env.example
└── README.md

../kind-config.yaml            # 1 control-plane + 2 worker Kind cluster definition
../k8s/
├── namespace.yaml
├── configmap.yaml             # Non-sensitive app config + DB init/migration SQL
├── secrets.yaml                # DB credentials
├── pvc.yaml                    # PersistentVolume + PersistentVolumeClaim (Postgres)
├── database-deployment.yaml
├── backend-deployment.yaml
├── frontend-deployment.yaml
├── services.yaml
└── ingress.yaml
```

## Technology Stack

| Component | Technology |
|-----------|------------|
| Frontend | React + Vite, served by Nginx |
| Backend | Node.js + Express |
| Database | PostgreSQL 16 |
| Orchestration | Kubernetes (Kind) |
| Ingress | NGINX Ingress Controller |

## Deploying

```bash
# 1. Create the cluster (1 control-plane + 2 workers)
kind create cluster --config kind-config.yaml

# 2. Install the NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller --timeout=180s

# 3. Build application images
docker build -t bmi-backend:v1 ./app/backend
docker build -t bmi-frontend:v1 ./app/frontend

# 4. Load images into the Kind cluster (no external registry needed)
kind load docker-image bmi-backend:v1 bmi-frontend:v1 --name three-tier

# 5. Apply the manifests
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/secrets.yaml -f k8s/configmap.yaml -f k8s/pvc.yaml
kubectl apply -f k8s/database-deployment.yaml
kubectl apply -f k8s/backend-deployment.yaml -f k8s/frontend-deployment.yaml -f k8s/services.yaml -f k8s/ingress.yaml
```

## Accessing the App

The Ingress Controller is pinned to the control-plane node, whose port 80/443 is mapped to the host via `kind-config.yaml`'s `extraPortMappings`.

**http://localhost/**

## Verifying the Deployment

```bash
kubectl get pods -n bmi-app -o wide          # pod distribution across nodes
kubectl get pvc,pv -n bmi-app                # storage bound
kubectl get svc,ingress -n bmi-app           # networking
curl http://localhost/api/measurements       # end-to-end request through the stack
```

## Configuration

- **ConfigMap** (`app-config`): `NODE_ENV`, `PORT`, `DB_HOST`, `DB_PORT`, `DB_NAME`, `CORS_ORIGIN`, `FRONTEND_URL`
- **Secret** (`db-secret`): `POSTGRES_USER`, `POSTGRES_PASSWORD`, `DB_USER`, `DB_PASSWORD`, `DATABASE_URL`

No credentials are hardcoded in the application code or Docker images — both tiers read configuration from environment variables injected via `envFrom`/`valueFrom` in their Deployment specs.
