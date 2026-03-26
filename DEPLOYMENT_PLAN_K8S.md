# CD Deployment Plan — Shopizer on Colima + Kubernetes

## Overview

```
GitHub Actions CI          GHCR                  GitHub Actions CD        Colima (k8s)
─────────────────    →    ──────────────    →    ──────────────────   →   ─────────────
build + test               Docker images          kubectl apply             running pods
push image                 ghcr.io/...            rolling update
```

---

## 1. Images produced by existing CI

| Repo | Image |
|---|---|
| shopizer (backend) | `ghcr.io/<org>/shopizer:<sha>` |
| shopizer-shop-reactjs | `ghcr.io/<actor>/shopizer-shop:latest` |
| shopizer-admin | `shopizer-admin:<sha>` (local only — needs push step added) |

---

## 2. Prerequisites

```bash
# Install Colima + kubectl
brew install colima kubectl

# Start Colima with Kubernetes
colima start --kubernetes --cpu 4 --memory 8

# Verify
kubectl get nodes
```

---

## 3. Kubernetes Manifests

Create a `k8s/` folder in the **shopizer** repo (single source of truth for infra).

### Structure
```
k8s/
├── namespace.yaml
├── backend/
│   ├── deployment.yaml
│   └── service.yaml
├── storefront/
│   ├── deployment.yaml
│   └── service.yaml
└── admin/
    ├── deployment.yaml
    └── service.yaml
```

---

### `k8s/namespace.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: shopizer
```

---

### `k8s/backend/deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shopizer-backend
  namespace: shopizer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: shopizer-backend
  template:
    metadata:
      labels:
        app: shopizer-backend
    spec:
      containers:
        - name: shopizer-backend
          image: ghcr.io/OWNER/shopizer:latest
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "local"
          readinessProbe:
            httpGet:
              path: /api/v1/store/DEFAULT
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "500m"
```

### `k8s/backend/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: shopizer-backend
  namespace: shopizer
spec:
  selector:
    app: shopizer-backend
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30080
  type: NodePort
```

---

### `k8s/storefront/deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shopizer-storefront
  namespace: shopizer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: shopizer-storefront
  template:
    metadata:
      labels:
        app: shopizer-storefront
    spec:
      containers:
        - name: shopizer-storefront
          image: ghcr.io/OWNER/shopizer-shop:latest
          ports:
            - containerPort: 80
          env:
            - name: APP_BASE_URL
              value: "http://localhost:30080"
            - name: APP_API_VERSION
              value: "/api/v1/"
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "200m"
```

### `k8s/storefront/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: shopizer-storefront
  namespace: shopizer
spec:
  selector:
    app: shopizer-storefront
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30300
  type: NodePort
```

---

### `k8s/admin/deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shopizer-admin
  namespace: shopizer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: shopizer-admin
  template:
    metadata:
      labels:
        app: shopizer-admin
    spec:
      containers:
        - name: shopizer-admin
          image: ghcr.io/OWNER/shopizer-admin:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "200m"
```

### `k8s/admin/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: shopizer-admin
  namespace: shopizer
spec:
  selector:
    app: shopizer-admin
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30400
  type: NodePort
```

---

## 4. CD GitHub Actions Workflow

Add a `cd.yml` to each repo's `.github/workflows/`. Triggers after CI pushes the image.

### Required GitHub Secrets
```
KUBECONFIG_DATA    # base64-encoded ~/.kube/config from Colima
```

Get it:
```bash
cat ~/.kube/config | base64 | pbcopy
```

---

### `shopizer` — `.github/workflows/cd.yml`
```yaml
name: CD

on:
  workflow_run:
    workflows: ["CI"]
    branches: [main]
    types: [completed]

jobs:
  deploy:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set kubeconfig
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.KUBECONFIG_DATA }}" | base64 -d > ~/.kube/config

      - name: Deploy backend
        run: |
          kubectl set image deployment/shopizer-backend \
            shopizer-backend=ghcr.io/${{ github.repository }}:${{ github.sha }} \
            -n shopizer
          kubectl rollout status deployment/shopizer-backend -n shopizer --timeout=120s
```

---

### `shopizer-shop-reactjs` — `.github/workflows/cd.yml`
```yaml
name: CD

on:
  workflow_run:
    workflows: ["CI"]
    branches: [main]
    types: [completed]

jobs:
  deploy:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    steps:
      - name: Set kubeconfig
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.KUBECONFIG_DATA }}" | base64 -d > ~/.kube/config

      - name: Deploy storefront
        run: |
          kubectl set image deployment/shopizer-storefront \
            shopizer-storefront=ghcr.io/${{ github.actor }}/shopizer-shop:latest \
            -n shopizer
          kubectl rollout status deployment/shopizer-storefront -n shopizer --timeout=120s
```

---

### `shopizer-admin` — `.github/workflows/cd.yml`
```yaml
name: CD

on:
  workflow_run:
    workflows: ["CI"]
    branches: [main]
    types: [completed]

jobs:
  deploy:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    steps:
      - name: Set kubeconfig
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.KUBECONFIG_DATA }}" | base64 -d > ~/.kube/config

      - name: Deploy admin
        run: |
          kubectl set image deployment/shopizer-admin \
            shopizer-admin=ghcr.io/${{ github.actor }}/shopizer-admin:latest \
            -n shopizer
          kubectl rollout status deployment/shopizer-admin -n shopizer --timeout=120s
```

---

## 5. Fix needed in shopizer-admin CI

The admin CI builds the image locally but never pushes it to GHCR. Add this to the existing `ci.yml`:

```yaml
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Push Docker image
        run: |
          docker tag shopizer-admin:${{ github.sha }} ghcr.io/${{ github.actor }}/shopizer-admin:latest
          docker push ghcr.io/${{ github.actor }}/shopizer-admin:latest
```

---

## 6. Deployment Order (first time)

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/backend/
kubectl apply -f k8s/storefront/
kubectl apply -f k8s/admin/
```

---

## 7. Access URLs

| Service | URL |
|---|---|
| Backend API | `http://localhost:30080/api/v1/store/DEFAULT` |
| Storefront | `http://localhost:30300` |
| Admin | `http://localhost:30400` |

---

## 8. Rollback

```bash
kubectl rollout undo deployment/shopizer-backend -n shopizer
kubectl rollout undo deployment/shopizer-storefront -n shopizer
kubectl rollout undo deployment/shopizer-admin -n shopizer
```
