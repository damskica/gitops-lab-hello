# GitOps Lab: Argo CD + Kubernetes (Pull-based Continuous Deployment)

This repository demonstrates a practical **GitOps** workflow where:

- **Git** is the *single source of truth* for application and infrastructure state  
- configuration is **declarative** (YAML stored in Git)  
- **Argo CD** continuously compares *desired state* (Git) vs *live state* (cluster) and performs **pull-based reconciliation**  
- changes are delivered through **branch → pull request → review → merge**, providing a clear and auditable change history  

---

## What you’ll build

- A simple `hello` application (Kubernetes **Deployment** + **Service**) deployed into the `gitops-lab` namespace
- **Automated sync** from the `main` branch (Auto-Sync) with:
  - **Self-heal** (drift correction)
  - **Prune** (remove deleted resources)
- A drift demo: manually mutate the cluster and watch Argo CD restore the state from Git

---

## Prerequisites

### Tools
- `kubectl`
- `git`
- (optional) `gh` (GitHub CLI) for PR creation/merge
- Access to a Kubernetes cluster via kubeconfig

### Cluster access (jump host setup)
In this environment, `kubectl` runs on a **jump host** that has network access to the Kubernetes API.

If your kubeconfig is stored in a separate file (example: `~/.kube/config_prox`), export it:

```bash
export KUBECONFIG=~/.kube/config_prox
kubectl get nodes
```

---

## Repository layout

```
apps/
  hello/
    deploy.yaml
```

---

## Step 1 — Create the target namespace

```bash
kubectl create namespace gitops-lab
```

---

## Step 2 — Install Argo CD

> ⚠️ Argo CD installation creates CRDs and installs controllers into the cluster.

```bash
kubectl create namespace argocd

kubectl apply -n argocd -f   https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait until all pods are running:

```bash
kubectl get pods -n argocd -w
```

---

## Step 3 — (Optional) Access the Argo CD UI via SSH tunnel

If you run `kubectl` on a jump host, you can expose the Argo CD UI safely using `port-forward` on the jump host + an SSH tunnel from your laptop.

### 3.1 Port-forward on the jump host

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

### 3.2 SSH tunnel from your laptop

```bash
ssh -L 8080:127.0.0.1:8080 ubuntu@<jump-host>
```

Now open in a browser:

- `https://localhost:8080` (accept the self-signed certificate warning)

### 3.3 Get the admin password

Run on the jump host:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret   -o jsonpath="{.data.password}" | base64 -d; echo
```

Login:
- user: `admin`
- password: output of the command above

> You can complete the lab without the UI — everything can be done via `kubectl`.

---

## Step 4 — Define the application declaratively (Git = source of truth)

The app manifests live under `apps/hello/`.

Example `apps/hello/deploy.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  namespace: gitops-lab
spec:
  replicas: 2
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: hello
          image: nginx:1.28
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: hello
  namespace: gitops-lab
spec:
  selector:
    app: hello
  ports:
    - port: 80
      targetPort: 80
```

Commit and push to `main`:

```bash
git add apps/hello/deploy.yaml
git commit -m "GitOps: add hello manifests"
git push
```

---

## Step 5 — Create an Argo CD Application (pull-based deployment)

Create an `Application` custom resource in the `argocd` namespace that points Argo CD at this repo and path.

> Note: `repoURL` should be **HTTPS**. Your local git remote can still be SSH; Argo CD can read via HTTPS for public repositories.

Replace `<YOUR_GITHUB_USERNAME>/<YOUR_REPO>` with your repository:

```bash
kubectl apply -n argocd -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: hello
spec:
  project: default
  source:
    repoURL: https://github.com/<YOUR_GITHUB_USERNAME>/<YOUR_REPO>.git
    targetRevision: main
    path: apps/hello
  destination:
    server: https://kubernetes.default.svc
    namespace: gitops-lab
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

Check status:

```bash
kubectl -n argocd get application hello   -o jsonpath='{.status.sync.status}{" / "}{.status.health.status}'; echo
```

Verify resources exist in the cluster:

```bash
kubectl -n gitops-lab get deploy,svc,pods
```

---

## Step 6 — GitOps workflow: Branch → PR → Merge → Auto Deploy

1) Create a branch and change something (e.g., scale replicas or bump image):

```bash
git switch -c branch_test
# edit apps/hello/deploy.yaml
git add apps/hello/deploy.yaml
git commit -m "Scale hello to 3 replicas"
git push -u origin branch_test
```

2) Create and merge a Pull Request (optional `gh` CLI):

```bash
gh pr create --base main --head branch_test --fill
gh pr merge --squash --delete-branch
```

3) Argo CD will automatically detect the `main` update and deploy it.

Confirm live state:

```bash
kubectl -n gitops-lab get deploy hello   -o jsonpath='replicas={.spec.replicas} image={.spec.template.spec.containers[0].image}'; echo
```

---

## Step 7 — Demonstrate pull-based reconciliation (drift + self-heal)

Manually change the cluster (this simulates an out-of-band change):

```bash
kubectl -n gitops-lab scale deploy/hello --replicas=5
kubectl -n gitops-lab get deploy hello -o jsonpath='{.spec.replicas}'; echo
```

Because `selfHeal: true` is enabled, Argo CD will restore the value back to the one stored in Git.

Check again after a moment:

```bash
kubectl -n gitops-lab get deploy hello -o jsonpath='{.spec.replicas}'; echo
```

(Optional) Inspect Argo CD application events:

```bash
kubectl -n argocd describe application hello | sed -n '/Events:/,$p' | tail -n 50
```

---

## Rollback (GitOps-style)

To rollback a change, revert the commit (or PR merge commit) in Git:

```bash
git switch main
git pull
git revert <COMMIT_SHA>
git push
```

Argo CD will pull the reverted state and reconcile the cluster accordingly.

---

## Cleanup

Delete the Argo CD application:

```bash
kubectl -n argocd delete application hello
```

Delete the demo namespace:

```bash
kubectl delete namespace gitops-lab
```

(Optional) Uninstall Argo CD (removes the `argocd` namespace and its resources):

```bash
kubectl delete namespace argocd
```

---

## Notes / Troubleshooting

### Old revisions in Argo CD UI (rev1, rev2)
During rollouts, Kubernetes keeps old **ReplicaSets** (usually scaled to 0) for quick rollbacks.  
This behavior is controlled by `spec.revisionHistoryLimit` in the Deployment. Pods from old revisions are removed; the old ReplicaSet object may remain until the limit is exceeded.

### Private repository support
If the repo is private, you must add repo credentials in Argo CD (repository secret or SSH deploy key) so Argo CD can fetch manifests.
