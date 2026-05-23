# GitOps Example — Argo CD Practice Repo

A minimal GitOps repository for CKA / Argo CD practice.
Uses **Kustomize** base + overlays pattern with two environments: `dev` and `prod`.

---

## Repository Structure

```
gitops-example/
├── apps/
│   └── guestbook/
│       ├── base/                    # shared base manifests
│       │   ├── namespace.yaml
│       │   ├── configmap.yaml
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   └── kustomization.yaml
│       └── overlays/
│           ├── dev/                 # 1 replica, dev config
│           │   ├── kustomization.yaml
│           │   ├── patch-replicas.yaml
│           │   └── patch-configmap.yaml
│           └── prod/                # 3 replicas, higher resources
│               ├── kustomization.yaml
│               ├── patch-replicas.yaml
│               ├── patch-resources.yaml
│               └── patch-configmap.yaml
└── argocd/
    ├── app-dev.yaml                 # Argo CD Application — dev
    └── app-prod.yaml                # Argo CD Application — prod
```

---

## Step 1 — Push to GitHub

Argo CD needs a remote Git URL. Create a repo on GitHub and push:

```bash
cd /Users/pongsak.khamdee/repos/cka/gitops-example

# Replace with your GitHub username
git remote add origin https://github.com/<YOUR-USERNAME>/gitops-example.git
git push -u origin main
```

---

## Step 2 — Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all pods
kubectl wait --for=condition=Ready pod --all -n argocd --timeout=120s
```

---

## Step 3 — Access Argo CD UI

```bash
# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Open: https://localhost:8080
# Username: admin
# Password: (from above)
```

---

## Step 4 — Register the Git Repository

```bash
# Login via CLI
argocd login localhost:8080 --username admin --password <password> --insecure

# Add your repo (public — no credentials needed)
argocd repo add https://github.com/<YOUR-USERNAME>/gitops-example.git

# For private repo (use token)
argocd repo add https://github.com/<YOUR-USERNAME>/gitops-example.git \
  --username <YOUR-USERNAME> \
  --password <GITHUB-TOKEN>
```

---

## Step 5 — Deploy Applications

### Option A — kubectl apply

```bash
# Update repoURL in both files first!
sed -i 's|<YOUR-USERNAME>|your-actual-username|g' argocd/app-dev.yaml argocd/app-prod.yaml

kubectl apply -f argocd/app-dev.yaml
kubectl apply -f argocd/app-prod.yaml
```

### Option B — argocd CLI

```bash
# Dev
argocd app create guestbook-dev \
  --repo https://github.com/<YOUR-USERNAME>/gitops-example.git \
  --path apps/guestbook/overlays/dev \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace guestbook \
  --sync-policy automated \
  --auto-prune \
  --self-heal \
  --sync-option CreateNamespace=true

# Prod
argocd app create guestbook-prod \
  --repo https://github.com/<YOUR-USERNAME>/gitops-example.git \
  --path apps/guestbook/overlays/prod \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace guestbook \
  --sync-policy automated \
  --auto-prune \
  --self-heal \
  --sync-option CreateNamespace=true
```

---

## Step 6 — Verify

```bash
# Check Argo CD app status
argocd app list
argocd app get guestbook-dev
argocd app get guestbook-prod

# Check Kubernetes resources
kubectl get all -n guestbook

# Expected:
# dev:  1 replica  (dev-guestbook deployment)
# prod: 3 replicas (prod-guestbook deployment)
```

---

## Step 7 — Test GitOps Sync

Make a change and push — Argo CD auto-syncs within ~3 minutes:

```bash
# Change replica count in dev overlay
sed -i 's/replicas: 1/replicas: 2/' \
  apps/guestbook/overlays/dev/patch-replicas.yaml

git add . && git commit -m "scale dev to 2 replicas"
git push

# Watch Argo CD detect and apply the change
argocd app get guestbook-dev --watch
kubectl get pods -n guestbook -w
```

---

## Quick Reference

| Command | Purpose |
|---------|---------|
| `argocd app list` | List all apps and sync status |
| `argocd app sync guestbook-dev` | Manually trigger sync |
| `argocd app diff guestbook-dev` | Show diff between Git and cluster |
| `argocd app history guestbook-dev` | Show deployment history |
| `argocd app rollback guestbook-dev <id>` | Rollback to previous version |
| `argocd app delete guestbook-dev` | Delete app (and its resources) |

---

## Preview Kustomize Output Locally

```bash
# Requires: kubectl kustomize or kustomize CLI

# Dev overlay
kubectl kustomize apps/guestbook/overlays/dev

# Prod overlay
kubectl kustomize apps/guestbook/overlays/prod
```
