# Vprofile Kubernetes Deployment Guide

Complete guide to install all tools, deploy the app with Helm, and set up ArgoCD for GitOps.

---

## Table of Contents

1. [Install Required Tools](#1-install-required-tools)
2. [Start Minikube](#2-start-minikube)
3. [Deploy with Raw YAML](#3-option-a-deploy-with-raw-yaml-files)
4. [Deploy with Helm](#4-option-b-deploy-with-helm-recommended)
5. [Make Changes & Update](#5-make-changes--update)
6. [Rollout & Rollback](#6-rollout--rollback)
7. [Install & Setup ArgoCD](#7-install--setup-argocd)
8. [See Changes in ArgoCD](#8-see-changes-in-argocd)
9. [Push to GitHub](#9-push-to-github)
10. [Useful Commands](#10-useful-commands)

---

## 1. Install Required Tools

### Minikube (local Kubernetes cluster)

```bash
# Mac
brew install minikube

# Verify
minikube version
```

### kubectl (Kubernetes CLI)

```bash
# Mac
brew install kubectl

# Verify
kubectl version --client
```

### Helm (Kubernetes package manager)

```bash
# Mac
brew install helm

# Verify
helm version
```

### ArgoCD CLI

```bash
# Mac
brew install argocd

# Verify
argocd version
```

---

## 2. Start Minikube

```bash
# Start with enough resources
minikube start --cpus 2 --memory 4096

# Check it is running
minikube status
kubectl get nodes
```

---

## 3. Option A: Deploy with Raw YAML files

```bash
# Deploy all resources
kubectl apply -f vprofile-k8s/

# Check everything is running
kubectl get pods
kubectl get services

# Delete all resources (cleanup)
kubectl delete -f vprofile-k8s/
```

---

## 4. Option B: Deploy with Helm (Recommended)

The Helm chart is located at `helm/vprofile/`.

### First-time install

> If you previously deployed with `kubectl apply`, delete those resources first:
> ```bash
> kubectl delete -f vprofile-k8s/
> ```

```bash
# Install the chart
helm install vprofile helm/vprofile/

# Check the release
helm list

# Check pods are running
kubectl get pods
kubectl get services
```

### Access the app in browser

```bash
minikube service vproweb --url
```

### Uninstall

```bash
helm delete vprofile
```

---

## 5. Make Changes & Update

All configurable values are in `helm/vprofile/values.yaml`:

```yaml
db:
  image: abroor/vprofiledb:latest
  replicas: 1
  port: 3306
  rootPassword: ""           # keep empty here, use values-secret.yaml

memcache:
  image: memcached
  replicas: 1
  port: 11211

rabbitmq:
  image: rabbitmq
  replicas: 1
  port: 5672

app:
  image: abroor/vprofileapp:latest
  replicas: 1
  port: 8080

web:
  image: abroor/vprofileweb:latest
  replicas: 1
  port: 80
  nodePort: 30080
```

### Common changes

**Scale a service (e.g. web to 3 replicas):**
```yaml
web:
  replicas: 3
```

**Change an image tag:**
```yaml
app:
  image: abroor/vprofileapp:v2
```

**Change a port:**
```yaml
web:
  nodePort: 30090
```

### Apply changes after editing values.yaml

```bash
helm upgrade vprofile helm/vprofile/
```

Or use this single command that works for both first install and upgrades:

```bash
helm upgrade --install vprofile helm/vprofile/
```

### Preview changes before applying

```bash
# Render the templates without deploying
helm template vprofile helm/vprofile/

# Check for errors in the chart
helm lint helm/vprofile/
```

### Keeping secrets safe

Never put real passwords in `values.yaml` — it gets pushed to GitHub.

Create a local file `helm/vprofile/values-secret.yaml` (this file is in `.gitignore`):
```yaml
db:
  rootPassword: vprodbpass
```

Then deploy with both files:
```bash
helm upgrade --install vprofile helm/vprofile/ \
  -f helm/vprofile/values.yaml \
  -f helm/vprofile/values-secret.yaml
```

---

## 6. Rollout & Rollback

### Rollout (upgrade to a new version)

Every `helm upgrade` creates a new revision. Helm keeps full history.

```bash
# Make your change in values.yaml then run:
helm upgrade vprofile helm/vprofile/ --description "upgraded app to v2"

# Check pods are running the new version
kubectl get pods

# See full release history
helm history vprofile
```

`helm history vprofile` output:
```
REVISION  STATUS      DESCRIPTION
1         superseded  Install complete
2         deployed    upgraded app to v2
```

### Rollback to previous version

```bash
# Roll back to the previous revision
helm rollback vprofile

# Roll back to a specific revision number
helm history vprofile          # find the revision number first
helm rollback vprofile 1       # roll back to revision 1

# Verify rollback
kubectl get pods
helm history vprofile
```

The rolled-back release appears as a new revision in history — Helm never deletes history.

---

## 7. Install & Setup ArgoCD

ArgoCD watches your GitHub repo and auto-deploys when you push changes.

### Install ArgoCD in Minikube

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all pods to be ready (takes 1-2 minutes)
kubectl get pods -n argocd -w
```

### Access ArgoCD UI

```bash
# Forward the port
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open browser at: `https://localhost:8080`

Get the admin password:
```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Login with:
- **Username:** `admin`
- **Password:** output from the command above

### Connect your Helm chart to ArgoCD

The file `argocd-app.yaml` is already in the repo. Apply it:

```bash
kubectl apply -f vprofile-k8s/argocd-app.yaml
```

This tells ArgoCD to watch your GitHub repo and deploy from `vprofile-k8s/helm/vprofile/`.

Make sure `argocd-app.yaml` has the correct repo and branch:
```yaml
source:
  repoURL: https://github.com/Abrorjon77/vprofile-k8s.git
  targetRevision: master
  path: vprofile-k8s/helm/vprofile
```

---

## 8. See Changes in ArgoCD

### Workflow after pushing to GitHub

```
Edit values.yaml → git push → ArgoCD detects change → auto deploys
```

### ArgoCD checks GitHub every 3 minutes. To sync immediately:

**Via UI:** Open `https://localhost:8080` → click your app → click **Sync** → click **Synchronize**

**Via CLI:**
```bash
# Login
argocd login localhost:8080 --username admin --password <your-password> --insecure

# Sync now
argocd app sync vprofile

# Check app status
argocd app get vprofile
```

### App status meanings

| Status | Meaning |
|---|---|
| `OutOfSync` | New changes detected in GitHub, not deployed yet |
| `Syncing` | ArgoCD is applying changes now |
| `Synced` | Changes are live in the cluster |
| `Healthy` | All pods are running correctly |

### Verify changes applied

```bash
# Check pods restarted with new config
kubectl get pods

# Describe a pod to confirm new image
kubectl describe pod <pod-name>
```

---

## 9. Push to GitHub

```bash
# Check what changed
git status

# Stage your changes
git add .

# Commit
git commit -m "your message here"

# Push to GitHub
git push origin master
```

ArgoCD will detect the push within 3 minutes and auto-deploy.

---

## 10. Useful Commands

### Minikube

```bash
minikube start          # start the cluster
minikube stop           # stop the cluster
minikube status         # check status
minikube service --all  # open all services in browser
```

### kubectl

```bash
kubectl get pods                  # list all pods
kubectl get services              # list all services
kubectl get pods -w               # watch pods in real time
kubectl logs <pod-name>           # view pod logs
kubectl describe pod <pod-name>   # detailed pod info
kubectl get all                   # list everything
```

### Helm

```bash
helm list                                      # list releases
helm history vprofile                          # release history
helm install vprofile helm/vprofile/           # first install
helm upgrade vprofile helm/vprofile/           # apply changes
helm upgrade --install vprofile helm/vprofile/ # install or upgrade
helm rollback vprofile                         # roll back one revision
helm rollback vprofile 1                       # roll back to revision 1
helm delete vprofile                           # uninstall
helm lint helm/vprofile/                       # check for errors
helm template vprofile helm/vprofile/          # preview rendered YAML
```

### ArgoCD

```bash
argocd app list              # list all apps
argocd app get vprofile      # app details and status
argocd app sync vprofile     # force sync now
argocd app history vprofile  # deployment history
```
