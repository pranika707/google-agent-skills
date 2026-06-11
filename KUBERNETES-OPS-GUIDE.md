---
name: gke-kubectl
description: >
  Manage Google Kubernetes Engine (GKE) clusters using kubectl and gcloud.
  Use when the user wants to create GKE clusters, deploy workloads to Kubernetes,
  manage pods/deployments/services, set up Ingress, configure autoscaling, debug pods,
  manage namespaces, or work with Helm charts on GKE. Triggers on "GKE", "Kubernetes",
  "kubectl", "k8s", "cluster", "pod", "deployment", "Helm on GCP".
license: Apache-2.0
compatibility: Requires gcloud CLI, kubectl, and optionally Helm 3
metadata:
  author: google-agent-skills
  version: "1.0"
  gcp-services: GKE, Artifact Registry, Cloud Load Balancing, Cloud DNS
---

# GKE & kubectl

## Cluster Setup

```bash
# Create an Autopilot cluster (recommended — Google manages nodes)
gcloud container clusters create-auto my-cluster \
  --region us-central1

# Create a Standard cluster (more control)
gcloud container clusters create my-cluster \
  --region us-central1 \
  --num-nodes 3 \
  --machine-type e2-standard-4 \
  --enable-autoscaling \
  --min-nodes 1 \
  --max-nodes 10

# Get credentials (sets up kubectl context)
gcloud container clusters get-credentials my-cluster --region us-central1

# List clusters
gcloud container clusters list
```

## Deploying Workloads

### Deployment Manifest

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: us-central1-docker.pkg.dev/PROJECT_ID/my-repo/my-app:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        env:
        - name: NODE_ENV
          value: production
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
```

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/my-app
```

### Service & Ingress

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
---
# ingress.yaml (GKE Ingress — creates a Cloud Load Balancer)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    kubernetes.io/ingress.class: "gce"
    kubernetes.io/ingress.global-static-ip-name: "my-static-ip"
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /*
        pathType: ImplementationSpecific
        backend:
          service:
            name: my-app-service
            port:
              number: 80
```

## Common kubectl Operations

```bash
# Pods
kubectl get pods -n my-namespace
kubectl describe pod my-pod-xyz
kubectl logs my-pod-xyz --follow
kubectl exec -it my-pod-xyz -- /bin/bash

# Deployments
kubectl get deployments
kubectl scale deployment my-app --replicas=5
kubectl set image deployment/my-app my-app=IMAGE:v2
kubectl rollout undo deployment/my-app        # Rollback
kubectl rollout history deployment/my-app

# Services & Ingress
kubectl get services
kubectl get ingress
kubectl port-forward service/my-app-service 8080:80  # Local tunnel

# ConfigMaps & Secrets
kubectl create configmap app-config --from-file=config.yaml
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=s3cr3t

# Namespaces
kubectl create namespace staging
kubectl config set-context --current --namespace=staging
```

## Horizontal Pod Autoscaling

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

```bash
kubectl apply -f hpa.yaml
kubectl get hpa
```

## Workload Identity (Secure GCP Access)

```bash
# Bind Kubernetes SA to GCP SA (no key files needed)
gcloud iam service-accounts add-iam-policy-binding \
  my-gcp-sa@PROJECT_ID.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:PROJECT_ID.svc.id.goog[default/my-k8s-sa]"

kubectl annotate serviceaccount my-k8s-sa \
  iam.gke.io/gcp-service-account=my-gcp-sa@PROJECT_ID.iam.gserviceaccount.com
```

## Helm on GKE

```bash
# Add a chart repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install a chart
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace

# Upgrade
helm upgrade nginx-ingress ingress-nginx/ingress-nginx

# List releases
helm list -A
```

## Debugging

```bash
# Get cluster events
kubectl get events --sort-by='.lastTimestamp' -A

# Describe problematic pod
kubectl describe pod PODNAME

# Check resource usage
kubectl top nodes
kubectl top pods

# Get full YAML of any resource
kubectl get deployment my-app -o yaml
```

## Node Pools

```bash
# Add a GPU node pool
gcloud container node-pools create gpu-pool \
  --cluster my-cluster \
  --region us-central1 \
  --num-nodes 1 \
  --machine-type n1-standard-4 \
  --accelerator type=nvidia-tesla-t4,count=1

# Cordon a node (stop scheduling new pods)
kubectl cordon NODE_NAME

# Drain before maintenance
kubectl drain NODE_NAME --ignore-daemonsets --delete-emptydir-data
```
