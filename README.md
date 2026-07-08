# ArgoCD Sync Demo — manual sync (UI + CLI), then auto sync

One sample app, three phases:
  PHASE 0  install ArgoCD + register the app (manual-sync mode)
  PHASE 1  change -> git push -> app shows OutOfSync -> YOU sync (UI, then CLI)
  PHASE 2  switch the SAME app to automated sync -> push -> it syncs itself

```
argocd-sync-demo/
├── app/                            the sample app (nginx + version page)
└── argocd/
    ├── application-manual.yaml     Phase 1: no automated block  -> manual sync
    └── application-auto.yaml       Phase 2: automated+prune+selfHeal -> auto sync
```

=====================================================================
PHASE 0 — SETUP (once)
=====================================================================

## 0.1 Install ArgoCD (skip if already installed)
    kubectl create namespace argocd
    kubectl apply -n argocd --server-side --force-conflicts \
      -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s

## 0.2 UI access (leave running in its OWN terminal)
    kubectl -n argocd get secret argocd-initial-admin-secret \
      -o jsonpath="{.data.password}" | base64 -d; echo        # <- admin password
    kubectl port-forward svc/argocd-server -n argocd 8080:443
    # open https://localhost:8080  ->  user: admin

## 0.3 CLI login (needed for `argocd app sync` later)
    argocd login localhost:8080 --username admin --password '<PASTE>' --insecure

## 0.4 Push this folder to YOUR GitHub repo
    cd argocd-sync-demo
    git init && git add . && git commit -m "Version 1"
    git branch -M main
    git remote add origin https://github.com/YOUR-USERNAME/argocd-sync-demo.git
    git push -u origin main
    # then EDIT both argocd/application-*.yaml -> set repoURL to your repo

## 0.5 Register the app in MANUAL mode
    kubectl apply -f argocd/application-manual.yaml
    # First-ever deploy also needs one sync (manual mode never applies alone):
    argocd app sync sync-demo            # or click SYNC in the UI
    kubectl get pods -n sync-demo        # web pod Running

## 0.6 Open the sample app in the browser (keep the tab)
    minikube service web -n sync-demo --url     # open printed URL -> blue "Version 1"

=====================================================================
PHASE 1 — MANUAL SYNC (change -> push -> YOU trigger the sync)
=====================================================================

## 1.1 Make a change and push it
    sed -i 's/Version 1/Version 2/; s/#2563eb/#16a34a/' app/configmap.yaml
    sed -i 's/configVersion: "1"/configVersion: "2"/'   app/deployment.yaml
    git commit -am "Version 2 (green)" && git push

## 1.2 See ArgoCD DETECT but NOT act
    # UI: app card turns YELLOW "OutOfSync" (click REFRESH to check immediately)
    argocd app get sync-demo             # Sync Status: OutOfSync
    kubectl get pods -n sync-demo        # unchanged — nothing applied yet
    # Browser: still blue Version 1. Manual mode = detection only.

## 1.3a Sync via the UI
    #  open the app card -> press SYNC -> SYNCHRONIZE
    #  watch resources apply, pod rolls, card turns green "Synced"

## 1.3b (next change) Sync via the CLI instead
    sed -i 's/Version 2/Version 3/; s/#16a34a/#dc2626/' app/configmap.yaml
    sed -i 's/configVersion: "2"/configVersion: "3"/'   app/deployment.yaml
    git commit -am "Version 3 (red)" && git push

    argocd app diff sync-demo            # optional: preview what will change
    argocd app sync sync-demo            # THE manual CLI sync
    argocd app wait sync-demo --health   # block until Synced + Healthy

## 1.4 Verify in the browser
    # refresh the app tab (Ctrl+Shift+R) -> Version 2 (green), then 3 (red)

=====================================================================
PHASE 2 — AUTO SYNC (dynamic: push and do nothing)
=====================================================================

## 2.1 Flip the SAME app to automated (just apply the other manifest)
    kubectl apply -f argocd/application-auto.yaml
    argocd app get sync-demo | grep -A3 'Sync Policy'    # now Automated

## 2.2 Push a change — and DON'T touch the cluster
    sed -i 's/Version 3/Version 4/; s/#dc2626/#f59e0b/' app/configmap.yaml
    sed -i 's/configVersion: "3"/configVersion: "4"/'   app/deployment.yaml
    git commit -am "Version 4 (amber)" && git push

## 2.3 Watch it sync ITSELF (within ~3 min poll; Refresh in UI to force check)
    kubectl get pods -n sync-demo -w     # pod rolls with no sync command from you
    # browser refresh -> amber "Version 4"

## 2.4 Bonus: selfHeal (Git always wins now)
    kubectl scale deployment web -n sync-demo --replicas=5
    kubectl get pods -n sync-demo -w     # ArgoCD reverts to 1 within seconds

=====================================================================
CHEAT SHEET
=====================================================================
  MANUAL mode  = syncPolicy WITHOUT `automated` -> detects (OutOfSync), waits for you
                   UI:  app card -> SYNC -> SYNCHRONIZE
                   CLI: argocd app sync sync-demo
  AUTO mode    = syncPolicy WITH `automated{prune,selfHeal}` -> applies every push
  Switch modes = apply the other application-*.yaml (same app name -> updates in place)
  Useful CLI   = argocd app list | get | diff | sync | wait | history | rollback
  Cleanup      = kubectl delete application sync-demo -n argocd
