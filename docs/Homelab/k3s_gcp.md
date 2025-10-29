# K3s on GCP - Complete Setup Guide

## Table of Contents
1. [Cost Overview](#cost-overview)
2. [Single Node Setup (FREE)](#single-node-setup-free)
3. [3-Node Cluster Setup](#3-node-cluster-setup)
4. [Flask Application Example](#flask-application-example)
5. [Deployment Without Helm](#deployment-without-helm)
6. [Useful Commands](#useful-commands)

---

## Cost Overview

### GCP Free Tier
- **1 e2-micro VM instance** - FREE forever (US regions)
- **30 GB standard persistent disk** - FREE
- **1 GB network egress** - FREE (North America)

### Cost Estimate for 3-Node Cluster
| Component | Quantity | Monthly Cost |
|-----------|----------|--------------|
| e2-micro #1 (server) | 1 | $0 (free tier) |
| e2-micro #2 (agent) | 1 | ~$7-8 |
| e2-micro #3 (agent) | 1 | ~$7-8 |
| Standard disk (60GB) | 60GB | ~$1-2 |
| **TOTAL** | | **~$15-18/month** |

### Recommendation
**Use single node for learning** - it's 100% FREE and covers 90% of learning needs!

---

## Single Node Setup (FREE)

### 1. Create VM Instance

```bash
gcloud compute instances create k3s-free \
  --zone=us-west1-b \
  --machine-type=e2-micro \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=30GB \
  --boot-disk-type=pd-standard \
  --tags=k3s
```

### 2. Configure Firewall

```bash
# K3s API Server
gcloud compute firewall-rules create k3s-api \
  --allow=tcp:6443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=k3s

# HTTP/HTTPS for applications
gcloud compute firewall-rules create k3s-http \
  --allow=tcp:80,tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=k3s
```

### 3. Install K3s

```bash
# SSH into the VM
gcloud compute ssh k3s-free --zone=us-west1-b

# Install K3s
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode=644

# Verify installation
sudo kubectl get nodes
```

### 4. Setup Local Access

```bash
# Copy kubeconfig from server
gcloud compute scp k3s-free:/etc/rancher/k3s/k3s.yaml ~/.kube/k3s-free --zone=us-west1-b

# Get server external IP
EXTERNAL_IP=$(gcloud compute instances describe k3s-free \
  --zone=us-west1-b \
  --format='get(networkInterfaces[0].accessConfigs[0].natIP)')

# Update kubeconfig with external IP
sed -i "s/127.0.0.1/$EXTERNAL_IP/g" ~/.kube/k3s-free

# Use the config
export KUBECONFIG=~/.kube/k3s-free
kubectl get nodes
```

### Cost Saving

```bash
# Stop instance when not using
gcloud compute instances stop k3s-free --zone=us-west1-b

# Start when needed
gcloud compute instances start k3s-free --zone=us-west1-b
```

---

## 3-Node Cluster Setup

### 1. Create VMs

```bash
# Server node
gcloud compute instances create k3s-server \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=20GB \
  --tags=k3s-server

# Agent nodes
for i in 1 2; do
  gcloud compute instances create k3s-agent-$i \
    --zone=us-central1-a \
    --machine-type=e2-medium \
    --image-family=ubuntu-2204-lts \
    --image-project=ubuntu-os-cloud \
    --boot-disk-size=20GB \
    --tags=k3s-agent
done
```

### 2. Configure Firewall

```bash
# K3s API server
gcloud compute firewall-rules create k3s-api \
  --allow=tcp:6443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=k3s-server

# Internal cluster communication
gcloud compute firewall-rules create k3s-internal \
  --allow=tcp:0-65535,udp:0-65535,icmp \
  --source-tags=k3s-server,k3s-agent \
  --target-tags=k3s-server,k3s-agent
```

### 3. Install K3s Server

```bash
# SSH into server
gcloud compute ssh k3s-server --zone=us-central1-a

# Install K3s
curl -sfL https://get.k3s.io | sh -s - server \
  --write-kubeconfig-mode=644 \
  --disable=traefik \
  --node-external-ip=$(curl -s ifconfig.me)

# Get node token
sudo cat /var/lib/rancher/k3s/server/node-token

# Get server internal IP
hostname -I | awk '{print $1}'
```

### 4. Join Agent Nodes

```bash
# SSH into each agent
gcloud compute ssh k3s-agent-1 --zone=us-central1-a

# Join cluster (replace with your values)
curl -sfL https://get.k3s.io | K3S_URL=https://<SERVER_INTERNAL_IP>:6443 \
  K3S_TOKEN=<NODE_TOKEN> sh -

# Repeat for k3s-agent-2
```

### 5. Verify Cluster

```bash
# On server node
sudo kubectl get nodes
```

---

## Flask Application Example

### Directory Structure
```
flask-k3s-demo/
├── app.py
├── requirements.txt
├── Dockerfile
└── flask-deployment.yaml
```

### app.py

```python
from flask import Flask, jsonify
import os
import socket

app = Flask(__name__)

@app.route('/')
def home():
    return jsonify({
        'message': 'Hello from K3s on GCP!',
        'hostname': socket.gethostname(),
        'version': '1.0.0'
    })

@app.route('/health')
def health():
    return jsonify({'status': 'healthy'}), 200

@app.route('/info')
def info():
    return jsonify({
        'pod_name': socket.gethostname(),
        'pod_ip': socket.gethostbyname(socket.gethostname()),
        'app_version': '1.0.0'
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### requirements.txt

```
Flask==3.0.0
gunicorn==21.2.0
```

### Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
```

### Build and Push Image

```bash
# Set project ID
PROJECT_ID=$(gcloud config get-value project)

# Build image
docker build -t gcr.io/$PROJECT_ID/flask-k3s-app:v1.0.0 .

# Configure Docker
gcloud auth configure-docker

# Push to Google Container Registry
docker push gcr.io/$PROJECT_ID/flask-k3s-app:v1.0.0
```

---

## Deployment Without Helm

### flask-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
  labels:
    app: flask-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
      - name: flask-app
        image: gcr.io/YOUR_PROJECT_ID/flask-k3s-app:v1.0.0
        ports:
        - containerPort: 5000
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: flask-app-service
spec:
  type: LoadBalancer
  selector:
    app: flask-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5000
```

### Deploy Application

```bash
# Update YAML with your project ID
PROJECT_ID=$(gcloud config get-value project)
sed -i "s/YOUR_PROJECT_ID/$PROJECT_ID/g" flask-deployment.yaml

# Deploy
kubectl apply -f flask-deployment.yaml

# Check status
kubectl get pods
kubectl get svc

# Get external IP
kubectl get svc flask-app-service
```

### Test Application

```bash
# Get external IP
EXTERNAL_IP=$(kubectl get svc flask-app-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Test endpoints
curl http://$EXTERNAL_IP/
curl http://$EXTERNAL_IP/health
curl http://$EXTERNAL_IP/info
```

---

## Useful Commands

### Cluster Management

```bash
# Get nodes
kubectl get nodes

# Get all resources
kubectl get all -A

# Cluster info
kubectl cluster-info

# Node details
kubectl describe node <node-name>
```

### Pod Management

```bash
# List pods
kubectl get pods

# Pod details
kubectl describe pod <pod-name>

# View logs
kubectl logs <pod-name>

# Follow logs
kubectl logs -f <pod-name>

# Logs for all pods with label
kubectl logs -l app=flask-app --tail=50

# Execute command in pod
kubectl exec -it <pod-name> -- /bin/bash
```

### Deployment Management

```bash
# Scale deployment
kubectl scale deployment flask-app --replicas=5

# Update image (rolling update)
kubectl set image deployment/flask-app flask-app=gcr.io/$PROJECT_ID/flask-k3s-app:v1.1.0

# Check rollout status
kubectl rollout status deployment/flask-app

# Rollout history
kubectl rollout history deployment/flask-app

# Rollback to previous version
kubectl rollout undo deployment/flask-app

# Rollback to specific revision
kubectl rollout undo deployment/flask-app --to-revision=2
```

### Service Management

```bash
# List services
kubectl get svc

# Service details
kubectl describe svc flask-app-service

# Get endpoints
kubectl get endpoints
```

### Resource Management

```bash
# Apply configuration
kubectl apply -f deployment.yaml

# Delete resources
kubectl delete -f deployment.yaml

# Delete specific resource
kubectl delete deployment flask-app
kubectl delete service flask-app-service

# Edit resource
kubectl edit deployment flask-app
```

### Debugging

```bash
# Get events
kubectl get events --sort-by=.metadata.creationTimestamp

# Check resource usage
kubectl top nodes
kubectl top pods

# Port forward to local machine
kubectl port-forward service/flask-app-service 8080:80

# Describe issues
kubectl describe pod <pod-name>
```

### Namespace Management

```bash
# List namespaces
kubectl get namespaces

# Create namespace
kubectl create namespace dev

# Deploy to specific namespace
kubectl apply -f deployment.yaml -n dev

# Set default namespace
kubectl config set-context --current --namespace=dev
```

### Configuration

```bash
# View current context
kubectl config current-context

# View all contexts
kubectl config get-contexts

# Switch context
kubectl config use-context <context-name>

# View kubeconfig
kubectl config view
```

---

## Troubleshooting

### Pod Not Starting

```bash
# Check pod status
kubectl describe pod <pod-name>

# Check logs
kubectl logs <pod-name>

# Check events
kubectl get events
```

### Service Not Accessible

```bash
# Check service
kubectl get svc
kubectl describe svc flask-app-service

# Check endpoints
kubectl get endpoints flask-app-service

# Check firewall rules
gcloud compute firewall-rules list
```

### Image Pull Errors

```bash
# Verify image exists
gcloud container images list --repository=gcr.io/$PROJECT_ID

# Check if cluster can access GCR
kubectl get pods -o yaml | grep imagePullSecrets
```

### Node Issues

```bash
# Check node status
kubectl get nodes
kubectl describe node <node-name>

# SSH into node
gcloud compute ssh <node-name> --zone=<zone>

# Check K3s service
sudo systemctl status k3s
sudo systemctl status k3s-agent
```

---

## Cleanup

### Delete Application

```bash
kubectl delete -f flask-deployment.yaml
```

### Delete Cluster (Single Node)

```bash
gcloud compute instances delete k3s-free --zone=us-west1-b
gcloud compute firewall-rules delete k3s-api k3s-http
```

### Delete Cluster (3-Node)

```bash
gcloud compute instances delete k3s-server k3s-agent-1 k3s-agent-2 --zone=us-central1-a
gcloud compute firewall-rules delete k3s-api k3s-internal
```

---

## Additional Resources

- **K3s Documentation**: https://docs.k3s.io/
- **Kubernetes Documentation**: https://kubernetes.io/docs/
- **GCP Free Tier**: https://cloud.google.com/free
- **GCP Pricing Calculator**: https://cloud.google.com/products/calculator

---

## Notes

- Always use US regions (us-west1, us-central1, us-east1) for free tier eligibility
- Single e2-micro node is sufficient for learning Kubernetes/K3s
- Stop instances when not in use to avoid egress charges
- Use `kubectl get all -A` regularly to monitor cluster state
- Keep your kubeconfig file secure
- Consider using preemptible instances for even lower costs (not eligible for free tier)

---

**Last Updated**: October 2025