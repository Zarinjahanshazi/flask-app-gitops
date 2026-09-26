# Argo CD on Amazon EKS — GitOps Deployment Pipeline

[![Status](https://img.shields.io/badge/status-complete-brightgreen)]()
[![Kubernetes](https://img.shields.io/badge/kubernetes-1.33-326CE5?logo=kubernetes&logoColor=white)]()
[![Argo CD](https://img.shields.io/badge/GitOps-Argo%20CD-EF7B4D?logo=argo&logoColor=white)]()
[![AWS](https://img.shields.io/badge/AWS-EKS-FF9900?logo=amazonaws&logoColor=white)]()

A production-style **GitOps pipeline** on Amazon EKS. Two custom applications
— a Flask backend and a Next.js frontend — are continuously deployed and kept
in sync by **Argo CD**, which watches a private Git repository of Kubernetes
manifests. A **GitHub Actions** pipeline builds and pushes new container
images on every push to `main`, so the path from code change to running pod
is fully automated — no manual `kubectl apply` or `docker push` required.

> Built as a personal, hands-on project — designed end-to-end (custom apps,
> own infrastructure) to learn GitOps patterns in depth, not just follow a
> tutorial.

---

## Table of contents

- [Architecture](#architecture)
- [Applications](#applications)
- [Repository layout](#repository-layout)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [CI/CD pipeline](#cicd-pipeline)
- [Monitoring](#monitoring)
- [Command reference](#command-reference)
- [Troubleshooting notes](#troubleshooting-notes)
- [Teardown](#teardown)
- [Roadmap](#roadmap)

---

## Architecture

```
Developer pushes code
      │
      ▼
GitHub Actions CI   (build image → push to Docker Hub → bump tag in GitOps repo)
      │
      ├──────────────► Docker Hub          (image storage)
      │
      ▼
GitOps repo (private) — Kubernetes manifests
      │
      ▼
Argo CD   (watches the GitOps repo, auto-syncs)
      │
      ▼
Amazon EKS cluster
   ├── flask-app namespace    — backend pods
   ├── nextjs-app namespace   — frontend pods
   └── monitoring namespace   — Prometheus + Grafana
      │
      ▼
End users (via LoadBalancer)
```

**Core principle:** Argo CD watches **Git**, not the container registry.
Pushing a new image alone triggers nothing — a commit to the GitOps repo is
the only thing that starts a rollout. This separation (build vs. deploy) is
what makes the pipeline auditable and reproducible.

---

## Applications

| App | Stack | Port | Notes |
|---|---|---|---|
| `flask-basic-app` | Python / Flask, served by Gunicorn | `5007` | `/` returns JSON info, `/healthz` is the liveness/readiness probe. CORS enabled via `flask-cors`. Runs as a non-root user (UID 1000). |
| `nextjs-app` | Next.js (App Router) | `3000` | "Check backend" button fetches the backend's `/` and renders the JSON response. Built with `output: 'standalone'` via a multi-stage Dockerfile. Runs as `node:20-slim`'s built-in non-root `node` user. |

---

## Repository layout

This project spans three repositories:

| Repository | Purpose | Visibility |
|---|---|---|
| `flask-basic-app` | Backend source, Dockerfile, CI workflow | Public |
| `nextjs-app` | Frontend source, Dockerfile, CI workflow | Public |
| `flask-app-gitops` | Kubernetes manifests + Argo CD `Application` definitions — **this is what Argo CD watches** | Private |

```
flask-app-gitops/
├── apps/
│   ├── flask-basic-app/
│   │   └── k8s/
│   │       ├── namespace.yaml
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       ├── hpa.yaml
│   │       └── pdb.yaml
│   └── nextjs-app/
│       └── k8s/
│           └── ... (same structure)
└── argocd-apps/
    ├── flask-basic-app-application.yaml
    └── nextjs-app-application.yaml
```

**Namespaces in the cluster:** `flask-app`, `nextjs-app`, `argocd`, `monitoring`.

---

## Prerequisites

- AWS account with credentials configured (`aws sts get-caller-identity` working)
- `eksctl`, `kubectl`, `helm` installed
- A Docker Hub account (or any container registry)
- A fine-grained GitHub PAT with `Contents: Read and write` scoped to the GitOps repo

---

## Setup

### 1. Create the EKS cluster

```bash
eksctl create cluster \
  --name graaho-eks \
  --region ap-south-1 \
  --version 1.33 \
  --nodegroup-name graaho-ng \
  --node-type t3.medium \
  --nodes 2 --nodes-min 2 --nodes-max 4 \
  --managed
```

### 2. Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Expose the Argo CD UI
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# Retrieve the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d
```

### 3. Connect the private GitOps repo

In the Argo CD UI: **Settings → Repositories → Connect Repo**, using a
fine-grained GitHub PAT scoped to the `flask-app-gitops` repo
(`Contents: Read and write`). Write access is intentional — the same token
is reused as `GITOPS_TOKEN` by the CI pipeline in step 5.

### 4. Deploy the applications

```bash
kubectl apply -f argocd-apps/flask-basic-app-application.yaml
kubectl apply -f argocd-apps/nextjs-app-application.yaml
```

Both apps should reach **Healthy / Synced** in the Argo CD UI. Fetch each
service's public LoadBalancer hostname:

```bash
kubectl get svc flask-basic-app -n flask-app \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
kubectl get svc nextjs-app -n nextjs-app \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

> ⚠️ **Important:** rebuild and push the `nextjs-app` image with
> `--build-arg NEXT_PUBLIC_BACKEND_URL=http://<backend-elb-hostname>` once the
> backend's LoadBalancer hostname is known — the default value
> (`localhost:5007`) only works for local testing, not for real visitors.
> See [Troubleshooting notes](#troubleshooting-notes).

---

## CI/CD pipeline

Each source repo (`flask-basic-app`, `nextjs-app`) has a
`.github/workflows/deploy.yaml` that, on every push to `main`:

1. Checks out the source and computes a short-SHA image tag
2. Logs in to Docker Hub and builds/pushes a `linux/amd64` image
3. Checks out the private GitOps repo using `GITOPS_TOKEN`
4. Patches the image tag in the relevant `deployment.yaml`
5. Commits and pushes the change back to the GitOps repo — which Argo CD then syncs automatically

```yaml
name: Build, Push & Update GitOps

on:
  push:
    branches: [ main ]

env:
  IMAGE: <dockerhub-username>/flask-basic-app
  GITOPS_REPO: <github-username>/flask-app-gitops
  MANIFEST: apps/flask-basic-app/k8s/deployment.yaml

jobs:
  build-and-update:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout source code
        uses: actions/checkout@v4

      - name: Set image tag (short commit SHA)
        id: vars
        run: echo "sha=${GITHUB_SHA::7}" >> "$GITHUB_OUTPUT"

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build & push image (linux/amd64)
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          platforms: linux/amd64
          tags: ${{ env.IMAGE }}:${{ steps.vars.outputs.sha }}

      - name: Check out the GitOps config repo
        uses: actions/checkout@v4
        with:
          repository: ${{ env.GITOPS_REPO }}
          token: ${{ secrets.GITOPS_TOKEN }}
          path: gitops

      - name: Bump image tag in the config repo
        working-directory: gitops
        run: |
          sed -i -E "s|(image: ${IMAGE}:).*|\1${{ steps.vars.outputs.sha }}|" "$MANIFEST"

      - name: Commit & push to the config repo
        working-directory: gitops
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add "$MANIFEST"
          git commit -m "ci: flask-basic-app -> ${{ steps.vars.outputs.sha }}" || echo "no changes"
          git push
```

The `nextjs-app` workflow is identical except for its `env:` block and one
extra `build-args` field, which permanently bakes in the backend's public URL
so future CI-built images never regress to `localhost:5007`:

```yaml
env:
  IMAGE: <dockerhub-username>/nextjs-app
  GITOPS_REPO: <github-username>/flask-app-gitops
  MANIFEST: apps/nextjs-app/k8s/deployment.yaml
```

```yaml
      - name: Build & push image (linux/amd64)
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          platforms: linux/amd64
          build-args: |
            NEXT_PUBLIC_BACKEND_URL=http://<backend-elb-hostname>
          tags: ${{ env.IMAGE }}:${{ steps.vars.outputs.sha }}
```

### Required repository secrets

Set these under **Settings → Secrets and variables → Actions** on both source repos:

| Secret | Description |
|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub access token (Read & Write) |
| `GITOPS_TOKEN` | Fine-grained GitHub PAT scoped to the GitOps repo, `Contents: read/write` |

Images are tagged with the short commit SHA (never `:latest`), so every
deployed version traces back to an exact commit.

---

## Monitoring

Prometheus and Grafana are installed via the `kube-prometheus-stack` Helm chart:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create namespace monitoring
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.service.type=LoadBalancer
```

```bash
# Grafana admin password
kubectl get secret monitoring-grafana -n monitoring \
  -o jsonpath='{.data.admin-password}' | base64 -d

# Grafana LoadBalancer hostname
kubectl get svc monitoring-grafana -n monitoring \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

The built-in **Node Exporter Full** and **Kubernetes Networking / Cluster**
dashboards surface live per-namespace metrics out of the box.

> Note: monitoring is installed directly via Helm, not managed as an Argo CD
> `Application` — see [Roadmap](#roadmap) for a planned improvement.

---

## Command reference

```bash
# Cluster / pods
kubectl get nodes
kubectl get pods -A
kubectl get pods -A -o wide
kubectl get pods -n flask-app
kubectl get pods -n nextjs-app
kubectl get pods -n argocd
kubectl get pods -n monitoring

# Deployed image tag for a given deployment
kubectl get deployment <name> -n <namespace> \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# Manage Argo CD Applications without the argocd CLI
kubectl get applications -n argocd
kubectl delete application <name> -n argocd
```

---

## Troubleshooting notes

- **Frontend shows "Failed to fetch"** — the Next.js image was built with
  `NEXT_PUBLIC_BACKEND_URL` pointing at `localhost`, which only works for
  local Docker testing. Rebuild with `--build-arg
  NEXT_PUBLIC_BACKEND_URL=http://<real-backend-hostname>`, push, update the
  GitOps manifest, and let Argo CD sync.
- **`node:20-slim` already has a `node` user (UID 1000)** — don't `useradd`
  a new one with the same UID; reuse the existing user (`USER node`,
  `--chown=node:node`).
- **Chocolatey / Helm install on Windows fails with a lock-file error** — run
  PowerShell as Administrator.
- **`argocd` CLI not installed** — manage `Application` resources directly
  with `kubectl` instead (e.g. `kubectl delete application <name> -n argocd`).
- **Docker Hub UI shows only repo names** — check the **Tags** tab inside a
  repo to see individual image tags.
- On Windows, paste multi-line shell commands **one line at a time** —
  pasting several at once can merge them into a single broken command.

---

## Teardown

Delete resources in this order to avoid orphaned infrastructure and stop billing:

```bash
# 1. Argo CD Applications
kubectl delete application flask-basic-app -n argocd
kubectl delete application nextjs-app -n argocd

# 2. App namespaces
kubectl delete namespace flask-app nextjs-app

# 3. Monitoring stack
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring

# 4. Argo CD itself
kubectl delete namespace argocd

# 5. The cluster (this is what actually stops AWS billing)
eksctl delete cluster --name graaho-eks --region ap-south-1
```

After teardown, verify in the AWS Console: 0 EKS clusters, EC2 instances
terminated, 0 Load Balancers, 0 EBS volumes, and no leftover
`eksctl-<cluster-name>-*` CloudFormation stacks or VPCs.

---

## Roadmap

Ideas for a future iteration of this project:

- [ ] Move the monitoring stack into GitOps (Argo CD `Application` of `Helm` source type) instead of a manual `helm install`
- [ ] Replace the two separate LoadBalancers with a single ALB Ingress (AWS Load Balancer Controller) + TLS via ACM + Route 53 DNS
- [ ] Adopt an **App-of-Apps / ApplicationSet** pattern so new apps are added by dropping a file in Git
- [ ] Dedicated Argo CD `AppProject` instead of `project: default`, to restrict which repos/clusters/namespaces apps may touch
- [ ] Real Argo CD authentication (SSO/OIDC) instead of the default admin user
- [ ] IRSA (IAM Roles for Service Accounts) instead of broad node IAM roles
- [ ] Sealed Secrets or External Secrets Operator for any future in-cluster secrets

---

## License

This project is for educational/portfolio purposes. Feel free to fork and adapt.
