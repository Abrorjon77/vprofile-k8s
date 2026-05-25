# Vprofile Kubernetes Deployment Guide

## Prerequisites

- [Minikube](https://minikube.sigs.k8s.io/docs/start/) installed
- [kubectl](https://kubernetes.io/docs/tasks/tools/) installed
- [Helm](https://helm.sh/docs/intro/install/) installed

---

## Start Minikube

```bash
minikube start
```

---

## Option 1: Deploy with Raw YAML files

### Deploy all resources

```bash
kubectl apply -f /path/to/vprofile-k8s/
```

### Delete all resources

```bash
kubectl delete -f /path/to/vprofile-k8s/
```

---

## Option 2: Deploy with Helm (Recommended)

The Helm chart is located at `helm/vprofile/`.

### First-time install

> If you previously deployed with `kubectl apply`, delete those resources first:
> ```bash
> kubectl delete -f /path/to/vprofile-k8s/
> ```

Then install with Helm:

```bash
helm install vprofile helm/vprofile/
```

### Check the release

```bash
helm list
```

### Uninstall

```bash
helm delete vprofile
```

---

## How to Make Changes

All configurable values are in `helm/vprofile/values.yaml`:

```yaml
db:
  image: abroor/vprofiledb:latest
  replicas: 1
  port: 3306
  rootPassword: vprodbpass

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

---

## Apply Changes After Editing values.yaml

```bash
helm upgrade vprofile helm/vprofile/
```

Or use this single command that works for both first install and upgrades:

```bash
helm upgrade --install vprofile helm/vprofile/
```

---

## Preview Changes Before Applying

```bash
# Render the templates without deploying
helm template vprofile helm/vprofile/

# Check for errors in the chart
helm lint helm/vprofile/
```

---

## Check Deployment Status

```bash
# List all pods
kubectl get pods

# List all services
kubectl get services

# Watch pods in real time
kubectl get pods -w

# View logs for a pod
kubectl logs <pod-name>
```

---

## Access the App in Browser

```bash
minikube service vproweb --url
```

Or open all services at once:

```bash
minikube service --all
```

---

## Rollout (Upgrade to a New Version)

A rollout happens every time you run `helm upgrade`. Helm keeps a history of every release revision.

### Step 1 — make your change in values.yaml

For example, update the app image:
```yaml
app:
  image: abroor/vprofileapp:v2
```

### Step 2 — upgrade with a description (recommended)

```bash
helm upgrade vprofile helm/vprofile/ --description "upgraded app to v2"
```

### Step 3 — verify the rollout

```bash
# Check pods are running the new version
kubectl get pods

# See Helm release history
helm history vprofile
```

`helm history vprofile` output shows each revision:

```
REVISION  STATUS      DESCRIPTION
1         superseded  Install complete
2         deployed    upgraded app to v2
```

---

## Rollback to a Previous Version

### Roll back to the previous revision

```bash
helm rollback vprofile
```

### Roll back to a specific revision number

```bash
# First check history to find the revision number
helm history vprofile

# Then roll back to that revision (e.g. revision 1)
helm rollback vprofile 1
```

### Verify the rollback

```bash
# Check pods restarted with the old version
kubectl get pods

# Confirm current revision in history
helm history vprofile
```

The rolled-back release will appear as a new revision in history (Helm never deletes history).
