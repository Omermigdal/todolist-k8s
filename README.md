# Kubernetes Todolist Application Deployment

This guide explains how to deploy the todolist application on a Kubernetes cluster using Kustomize for configuration management.

## Prerequisites

- A running Kubernetes cluster
- `kubectl` configured to communicate with your cluster
- `kustomize` installed (optional, but `kubectl apply -k` supports Kustomize natively)
- Sufficient cluster resources (see resource requirements below)

## Resource Requirements

- **MariaDB**: 512Mi-1Gi memory, 250m-500m CPU, persistent storage (configurable by overlay)
- **Todolist App**: 256Mi-512Mi memory, 100m-250m CPU per replica (2 replicas by default)

## Project Structure

```
.
├── base/                     # Base stack: MariaDB Deployment + services + app Deployment
│   ├── configmap.yaml        # todolist-config
│   ├── secret.yaml           # todolist-secret
│   ├── mariadb-deploy.yaml   # MariaDB Deployment (emptyDir by default)
│   ├── mariadb-svc.yaml      # backend ClusterIP service
│   ├── todolist-deploy.yaml  # todolist Deployment with init container
│   ├── todolist-svc.yaml     # LoadBalancer service
│   └── kustomization.yaml
├── overlays/
│   ├── 1-with-pvc/           # Adds PVC to MariaDB Deployment
│   ├── 2-with-sts-and-cj/    # Switches MariaDB to StatefulSet + headless svc + backup CronJob
│   ├── 3-with-hpa/           # Adds HPA for todolist app + restores app resource requests/limits
│   └── 4-with-probes/        # Adds app health probes; builds on 3-with-hpa
├── metallb-config.yaml       # MetalLB address-pool configuration for LoadBalancer
└── README.md
```

## Overlay Catalog

**Base (default)**

- MariaDB Deployment with `emptyDir`
- Backend ClusterIP service
- Todolist Deployment (init waits for DB), LoadBalancer service

**1-with-pvc**

- Patches MariaDB Deployment volume to PVC for data persistence

**2-with-sts-and-cj**

- Removes MariaDB Deployment; replaces with StatefulSet + headless service
- Adds backup CronJob and backup PVC

**3-with-hpa**

- Adds HPA for `todolist-app` (targets 50% CPU, scales 2→4)
- Re-applies app container resource requests/limits via patch
- Currently based on the base DB flavor (ephemeral); extend it if you need PVC/STS + HPA together

**4-with-probes**

- Builds on `3-with-hpa` and adds HTTP health probes to the app Deployment
- Readiness: checks `/health` on container port `80` (15s initial delay, every 5s, timeout 3s, fail 3)
- Liveness: checks `/health` on container port `80` (30s initial delay, every 10s, timeout 5s, fail 3)
- Also overrides app image to `ghcr.io/bennyro-mta/todolist:1.8`
- Note: adjust the probe port/path if your container listens on a different port (base uses `5000`).

## Deployment Instructions

### Step 1: Create Namespace (Optional)

```bash
kubectl create namespace todolist
# If using a namespace, add -n todolist to all subsequent commands
```

### Step 2: Deploy the Stack (choose one overlay)

Base (ephemeral DB storage):

```bash
kubectl apply -k base/
```

Add persistence (Deployment + PVC):

```bash
kubectl apply -k overlays/1-with-pvc/
```

StatefulSet + backup CronJob:

```bash
kubectl apply -k overlays/2-with-sts-and-cj/
```

Add HPA for the app (2→4 replicas at 50% CPU; re-applies app resource limits). This overlay includes the base stack (ephemeral DB); create a custom overlay if you need HPA plus PVC/STS.

```bash
kubectl apply -k overlays/3-with-hpa/
```

Add app health probes (inherits `3-with-hpa` and overrides the app image to 1.8):

```bash
kubectl apply -k overlays/4-with-probes/
```

## Complete Deployment Script

Deploy everything at once with the current overlays (examples):

```bash
#!/bin/bash
# Deploy common resources
kubectl apply -f secret.yaml
kubectl apply -f configmap.yaml

# Option A: Base (ephemeral DB)
kubectl apply -k base/

# Option B: DB with PVC (Deployment + PVC)
kubectl apply -k overlays/1-with-pvc/

# Option C: DB StatefulSet + backup CronJob
kubectl apply -k overlays/2-with-sts-and-cj/

# (Optional) App autoscaling (base stack with HPA on todolist)
kubectl apply -k overlays/3-with-hpa/

# (Optional) App health probes (inherits 3-with-hpa)
kubectl apply -k overlays/4-with-probes/

# Deploy MetalLB if needed
kubectl apply -f metallb-config.yaml
```

## Modifying MariaDB Configuration

### Changing Backup Schedule

Edit `overlays/2-with-sts-and-cj/backup.yaml` and update the `schedule` field in the CronJob spec:

```yaml
spec:
  schedule: "0 2 * * *" # Daily at 2 AM
```

### Changing Storage Size

For **1-with-pvc** overlay, edit `overlays/1-with-pvc/pvc.yaml`:

```yaml
storage: 5Gi # Change from 1Gi to desired size
```

For **2-with-sts-and-cj** overlay, edit `overlays/2-with-sts-and-cj/statefulset.yaml`:

```yaml
storage: 5Gi # Change in volumeClaimTemplates
```

For **2-with-sts-and-cj** overlay, edit both:

- `overlays/2-with-sts-and-cj/backup.yaml` (backup storage PVC)
- `overlays/2-with-sts-and-cj/statefulset.yaml` (data volumeClaimTemplates)

## Environment Variables

MariaDB configuration is managed through:

- **Secret** `todolist-secret`: Contains `MYSQL_ROOT_PASSWORD`
- **ConfigMap** `todolist-config`: Contains `MYSQL_DB`, `MYSQL_HOST`, `MYSQL_USER`

Ensure these resources are created in your cluster before deploying MariaDB.

## Troubleshooting

### MariaDB Pod Not Starting

Check logs:

```bash
kubectl logs -l app=mariadb
```

Check PVC status:

```bash
kubectl get pvc
kubectl describe pvc mariadb-pvc  # or mariadb-data-mariadb-0 for StatefulSet
```

### Backup CronJob Not Running

Check CronJob status:

```bash
kubectl get cronjob mariadb-backup
kubectl get jobs -l app=mariadb-backup
kubectl logs -l app=mariadb-backup
```

### Todolist Unable to Connect to Database

Verify ConfigMap has correct MYSQL_HOST:

```bash
kubectl get configmap todolist-config -o yaml
```

Check backend service:

```bash
kubectl get svc backend
```

## Verification — local smoke tests (works on any computer)

Quick smoke test (minikube example):

1. Start cluster + ingress

```bash
minikube start --driver=docker --cpus=2 --memory=4096
minikube addons enable ingress
```

2. Install the chart (provide root password)

```bash
helm upgrade --install my-release ./helm-chart --set secret.rootPassword=Password --wait --timeout 5m
```

3. Wait for pods to be ready

```bash
kubectl wait --for=condition=ready pod -l app=todos-api --timeout=120s
kubectl get pods -o wide
```

4. Expose ingress locally (preferred)

```bash
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80
```

If `localhost:8080` is already used, either free the port or forward to another local port (example 9090):

- Free 8080 (Windows):

```powershell
netstat -aon | findstr :8080
taskkill /PID <pid> /F
```

- Free 8080 (macOS/Linux):

```bash
lsof -i:8080
kill <pid>
```

- Or use alternate local port:

```bash
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 9090:80
```

5. Smoke requests (expect 200)

```bash
curl -fsS http://localhost:8080/api/todos/    # API
curl -fsS http://localhost:8080/              # frontend
```

(If you used 9090, replace the port accordingly.)

6. Tear down (when done)

```bash
helm uninstall my-release
minikube stop
```

Deterministic checks / CI notes

- Always set `--set secret.rootPassword=...` for installs and CI pipelines.
- Use fixed image tags (already set in `values.yaml`) to avoid flakes.
- If your environment has no StorageClass, either create one or override `mariadb.storageClass`.

Migration / backup note

- The chart now standardizes the MariaDB StatefulSet/service name on `mariadb.name`. Upgrades include compatibility checks, but if you have persistent data, back it up before changing storage or deleting PVCs.
- Quick backup example:

```bash
kubectl exec -it $(kubectl get pod -l app=mariadb -o jsonpath='{.items[0].metadata.name}') -- mysqldump -uroot -p$MARIADB_ROOT_PASSWORD $MYSQL_DB > /tmp/dump.sql
kubectl cp $(kubectl get pod -l app=mariadb -o jsonpath='{.items[0].metadata.name}'):/tmp/dump.sql ./dump.sql
```

CI suggestion

- Add a pipeline that creates a fresh cluster (kind/minikube), installs `ingress-nginx`, runs `helm lint`, installs the chart, runs the smoke curl tests above, and tears down the cluster.

## Troubleshooting

### Common Issues

1. **Pods stuck in Pending state**
   - Check if PVC can be bound: `kubectl get pvc`
   - Ensure sufficient cluster resources

2. **Database connection errors**
   - Verify secrets are correctly applied: `kubectl get secrets`
   - Check if MariaDB pod is running: `kubectl get pods -l app=mariadb`

3. **LoadBalancer service has no external IP**
   - Check if your cluster supports LoadBalancer services
   - Consider using NodePort or configuring MetalLB

### Useful Commands

```bash
# View detailed pod information
kubectl describe pod <pod-name>

# Access pod shell for debugging
kubectl exec -it <pod-name> -- /bin/bash

# View persistent volumes
kubectl get pv,pvc

# Check events for troubleshooting
kubectl get events --sort-by='.lastTimestamp'
```

## Other resources

- `metallb-config.yaml`: address-pool configuration for MetalLB in minikube. Apply it after enabling MetalLB (e.g., `minikube addons enable metallb`) so LoadBalancer services (like `todolist-svc`) receive an external IP. Without it, LoadBalancer services will stay pending.

## Cleanup

Delete the stack you applied (pick the same overlay/base you used):

```bash
# Base
kubectl delete -k base/

# Or: Deployment + PVC
kubectl delete -k overlays/1-with-pvc/

# Or: StatefulSet + backup CronJob
kubectl delete -k overlays/2-with-sts-and-cj/

# Or: App HPA (base stack with HPA)
kubectl delete -k overlays/3-with-hpa/

# Or: App HPA + health probes
kubectl delete -k overlays/4-with-probes/
```

PVCs are **not** deleted automatically. If you want to reclaim storage, delete the PVC(s) explicitly (this will delete data):

```bash
kubectl delete pvc --all
```

## Configuration

### Customizing the Deployment

- **Replicas**: Edit `todolist-deploy.yaml` to change the number of app replicas
- **Resources**: Adjust CPU/memory requests and limits in deployment files
- **Storage**: Modify `mariadb-pvc.yaml` to change storage size or class
- **Database credentials**: Update `secret.yaml` (remember to base64 encode values)

### Environment Variables

The application uses these environment variables (configured via ConfigMap and Secret):

- `MYSQL_HOST`: Database host (service name)
- `MYSQL_DB`: Database name
- `MYSQL_USER`: Database user
- `MYSQL_PASSWORD`: Database password
