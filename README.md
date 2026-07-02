# GitOps Demo — ArgoCD (local)

A tiny, pushable GitOps repo. You push this to GitHub, point ArgoCD at it, then
**edit a file → git push → watch the cluster update itself.** No `kubectl apply`.

```
gitops-demo/
├── app/                  <- what gets deployed (ArgoCD watches this folder)
│   ├── configmap.yaml    <- the editable HTML "code sample" (Version 1)
│   ├── deployment.yaml   <- nginx (public image), serves the HTML
│   └── service.yaml      <- NodePort so you can open it
└── argocd/
    └── application.yaml   <- the ArgoCD Application (edit repoURL to YOURS)
```

## Prereqs
- A local cluster running (minikube / kind / k3s / Docker Desktop)
- A **GitHub account** (a free public repo is fine)

---
## Part 1 — Install ArgoCD (once)

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
```

Get the admin password and open the UI (leave port-forward running in its own terminal):

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
kubectl port-forward svc/argocd-server -n argocd 8080:443
# open https://localhost:8080  (user: admin)
```

---
## Part 2 — Push THIS repo to your GitHub

Create an empty repo named `gitops-demo` on GitHub, then:

```bash
cd gitops-demo
git init
git add .
git commit -m "initial: Version 1"
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/gitops-demo.git
git push -u origin main
```

Now edit `argocd/application.yaml` and set `repoURL` to your repo URL.

---
## Part 3 — Tell ArgoCD to watch it

```bash
kubectl apply -f argocd/application.yaml
```

Watch it deploy (UI shows OutOfSync -> Synced/Healthy), or:

```bash
kubectl get pods -n gitops-demo -w
minikube service web -n gitops-demo --url    # open the URL -> "Version 1"
```

---
## Part 4 — THE SIMULATION: commit & push a change, cluster follows

### 4a. Change the content (your "code")
```bash
sed -i 's/Version 1/Version 2/' app/configmap.yaml
git commit -am "content: Version 2"
git push
```
Do NOTHING to the cluster. Within ~1 min ArgoCD sees the commit and syncs.
Refresh the web URL (or curl it) — it now shows **Version 2**.

```bash
kubectl get configmap web-content -n gitops-demo \
  -o jsonpath='{.data.index\.html}' | grep Version
```

### 4b. Instant visual: scale replicas
```bash
sed -i 's/replicas: 1/replicas: 3/' app/deployment.yaml
git commit -am "scale to 3"
git push
kubectl get pods -n gitops-demo -w     # watch 1 -> 3 pods, no kubectl apply
```

### 4c. Self-heal (Git always wins)
```bash
kubectl scale deployment web -n gitops-demo --replicas=10
# selfHeal: true reverts it back to what Git says (3) within seconds:
kubectl get pods -n gitops-demo -w
```

---
## Cleanup
```bash
kubectl delete -f argocd/application.yaml   # prune removes the app's resources
kubectl delete namespace argocd
```

## Mental model
`git push` is the ONLY action. ArgoCD watches Git and makes the cluster match —
new commits sync automatically, and manual drift is auto-reverted (selfHeal).
