# Ansible Tower (AWX) Kubernetes Operator — Installation Guide

## Overview

This document covers the full installation of the AWX Operator (open-source Ansible Tower) on a Kubernetes cluster, including the operator deployment, AWX instance, persistent storage, and an Ingress resource to expose the web console.

> **Note:** "Ansible Tower" is now called **AWX** in its open-source form. The AWX Operator is the official Kubernetes-native way to deploy it. The commercial product (Red Hat Ansible Automation Platform) uses the same operator pattern.

---

## Prerequisites

| Requirement         | Minimum Version | Notes                                      |
|---------------------|-----------------|--------------------------------------------|
| Kubernetes          | 1.24+           | EKS, GKE, AKS, k3s, kubeadm all supported |
| kubectl             | Matching cluster| Must have cluster-admin privileges         |
| Helm                | 3.x             | Used to install the operator               |
| StorageClass        | Any             | Default StorageClass must exist for PVCs   |
| Ingress Controller  | Any             | nginx-ingress recommended                  |
| DNS / Hostname      | —               | A hostname to expose the AWX console       |

### Verify Prerequisites

```bash
# Check cluster access
kubectl cluster-info

# Check kubectl version
kubectl version --client

# Check Helm version
helm version

# Confirm a default StorageClass exists
kubectl get storageclass

# Confirm an ingress controller is running
kubectl get pods -n ingress-nginx
```

---

## Step 1 — Create the AWX Namespace

```bash
kubectl create namespace awx
```

Set it as your default namespace for this guide (optional):

```bash
kubectl config set-context --current --namespace=awx
```

---

## Step 2 — Install the AWX Operator via Helm

### Add the AWX Operator Helm Repository

```bash
helm repo add awx-operator https://ansible.github.io/awx-operator/
helm repo update
```

### Verify Available Versions

```bash
helm search repo awx-operator
```

### Install the Operator

```bash
helm install awx-operator awx-operator/awx-operator \
  --namespace awx \
  --create-namespace \
  --set AWX.enabled=false
```

> `AWX.enabled=false` installs only the operator first. The AWX instance is deployed separately in Step 4, giving you full control over its configuration.

### Verify the Operator is Running

```bash
kubectl -n awx get pods
```

Expected output:
```
NAME                                               READY   STATUS    RESTARTS   AGE
awx-operator-controller-manager-xxxxxxxxx-xxxxx   2/2     Running   0          60s
```

---

## Step 3 — Create Persistent Volume Claims (Optional but Recommended)

If your StorageClass does not support dynamic provisioning, create PVCs manually.

### PostgreSQL PVC (`awx-postgres-pvc.yaml`)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: awx-postgres-13-pvc
  namespace: awx
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard   # Replace with your StorageClass name
  resources:
    requests:
      storage: 10Gi
```

### Projects PVC (`awx-projects-pvc.yaml`)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: awx-projects-pvc
  namespace: awx
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard   # Replace with your StorageClass name
  resources:
    requests:
      storage: 8Gi
```

Apply both:

```bash
kubectl apply -f awx-postgres-pvc.yaml
kubectl apply -f awx-projects-pvc.yaml
```

---

## Step 4 — Deploy the AWX Instance

Create the AWX custom resource. Save as `awx-instance.yaml`:

```yaml
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
  namespace: awx
spec:
  # --- Admin credentials ---
  admin_user: admin
  admin_email: admin@example.com       # Change this

  # --- Service type (ClusterIP since we use Ingress) ---
  service_type: ClusterIP

  # --- Hostname (must match your Ingress host below) ---
  hostname: awx.example.com            # Change this to your domain

  # --- PostgreSQL configuration ---
  postgres_storage_class: standard     # Change to your StorageClass
  postgres_storage_requirements:
    requests:
      storage: 10Gi

  # --- Projects persistent storage ---
  projects_persistence: true
  projects_storage_class: standard     # Change to your StorageClass
  projects_storage_size: 8Gi
  projects_existing_claim: awx-projects-pvc   # Remove if using dynamic provisioning

  # --- Resource requests/limits ---
  web_resource_requirements:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 1000m
      memory: 2Gi

  task_resource_requirements:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2000m
      memory: 2Gi

  # --- Extra settings (optional) ---
  extra_settings:
    - setting: SESSION_COOKIE_SECURE
      value: "true"
    - setting: CSRF_COOKIE_SECURE
      value: "true"
```

Apply the instance:

```bash
kubectl apply -f awx-instance.yaml
```

### Watch the Deployment Progress

```bash
kubectl -n awx get pods -w
```

The operator will create several pods. Full deployment takes **5–10 minutes**. Expected final state:

```
NAME                        READY   STATUS    RESTARTS   AGE
awx-operator-xxx            2/2     Running   0          10m
awx-postgres-13-0           1/1     Running   0          8m
awx-task-xxxxxxxxx-xxxxx    4/4     Running   0          5m
awx-web-xxxxxxxxx-xxxxx     3/3     Running   0          5m
```

### Check Operator Logs if Pods are Slow

```bash
kubectl -n awx logs -f deployment/awx-operator-controller-manager \
  -c awx-manager
```

---

## Step 5 — Retrieve the Auto-Generated Admin Password

```bash
kubectl -n awx get secret awx-admin-password \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

> Save this password. You will use it to log into the AWX console at **Step 7**.
> Username is: `admin`

---

## Step 6 — Create the Ingress Resource

### Option A — NGINX Ingress (HTTP only, for testing)

Save as `awx-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: awx-ingress
  namespace: awx
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
spec:
  ingressClassName: nginx
  rules:
    - host: awx.example.com             # Change to your domain
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: awx-service
                port:
                  number: 80
```

### Option B — NGINX Ingress with TLS (recommended for production)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: awx-ingress
  namespace: awx
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod   # Requires cert-manager
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - awx.example.com               # Change to your domain
      secretName: awx-tls-secret
  rules:
    - host: awx.example.com             # Change to your domain
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: awx-service
                port:
                  number: 80
```

Apply the ingress:

```bash
kubectl apply -f awx-ingress.yaml
```

### Verify the Ingress

```bash
kubectl -n awx get ingress
```

Expected output:
```
NAME          CLASS   HOSTS               ADDRESS        PORTS   AGE
awx-ingress   nginx   awx.example.com     10.0.0.x       80      30s
```

---

## Step 7 — DNS Configuration

Point your domain to the Ingress controller's external IP:

```bash
# Get the Ingress controller external IP
kubectl -n ingress-nginx get svc ingress-nginx-controller
```

Then create an A record in your DNS provider:

```
awx.example.com  →  <EXTERNAL-IP>
```

For local/dev testing without DNS, add to `/etc/hosts`:

```bash
echo "<EXTERNAL-IP>  awx.example.com" | sudo tee -a /etc/hosts
```

---

## Step 8 — Log Into the AWX Console

1. Open your browser and navigate to: `http://awx.example.com` or `https://awx.example.com`
2. Username: **`admin`**
3. Password: retrieved in Step 5

---

## Step 9 — Verify All Deployments

```bash
# All pods in awx namespace
kubectl -n awx get pods

# All services
kubectl -n awx get svc

# All PVCs
kubectl -n awx get pvc

# AWX custom resource status
kubectl -n awx get awx awx -o yaml | grep -A 20 "status:"

# Ingress
kubectl -n awx get ingress
```

---

## Troubleshooting

### AWX Web Pod Not Starting

```bash
kubectl -n awx describe pod <awx-web-pod-name>
kubectl -n awx logs <awx-web-pod-name> -c awx-web
```

### Database Connection Issues

```bash
kubectl -n awx logs <awx-task-pod-name> -c awx-task
kubectl -n awx logs awx-postgres-13-0
```

### Ingress Not Routing

```bash
# Confirm the service name matches the ingress backend
kubectl -n awx get svc

# Check ingress controller logs
kubectl -n ingress-nginx logs deployment/ingress-nginx-controller
```

### Reset Admin Password

```bash
kubectl -n awx exec -it deployment/awx-web -- awx-manage changepassword admin
```

### Operator Stuck / CrashLoopBackOff

```bash
kubectl -n awx describe pod <operator-pod>
helm -n awx uninstall awx-operator
helm install awx-operator awx-operator/awx-operator --namespace awx
```

---

## Upgrade the AWX Operator

```bash
helm repo update
helm upgrade awx-operator awx-operator/awx-operator --namespace awx
```

The operator will automatically reconcile and upgrade the AWX instance.

---

## Uninstall

```bash
# Remove AWX instance first
kubectl -n awx delete awx awx

# Remove the operator
helm -n awx uninstall awx-operator

# Remove the namespace (deletes everything)
kubectl delete namespace awx
```

---

## Summary of All Manifest Files

| File                    | Purpose                                      |
|-------------------------|----------------------------------------------|
| `awx-postgres-pvc.yaml` | PVC for PostgreSQL database storage          |
| `awx-projects-pvc.yaml` | PVC for Ansible project files                |
| `awx-instance.yaml`     | AWX custom resource (main deployment)        |
| `awx-ingress.yaml`      | Ingress to expose AWX console via hostname   |

---

## Key URLs and Credentials Reference

| Item             | Value                                                                        |
|------------------|------------------------------------------------------------------------------|
| Console URL      | `http(s)://awx.example.com`                                                  |
| Default Username | `admin`                                                                      |
| Default Password | `kubectl -n awx get secret awx-admin-password -o jsonpath="{.data.password}" | base64 -d` |
| Operator Logs    | `kubectl -n awx logs deploy/awx-operator-controller-manager -c awx-manager` |
| AWX Operator Repo| https://github.com/ansible/awx-operator                                      |
| AWX Docs         | https://ansible.readthedocs.io/projects/awx-operator                        |
