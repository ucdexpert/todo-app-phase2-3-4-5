# Minikube Deployment Specification

## Prerequisites Installation Steps

### 1. Install Docker
- Download and install Docker Desktop from [Docker's official website](https://www.docker.com/products/docker-desktop)
- Ensure Docker is running before proceeding
- Verify installation: `docker --version`

### 2. Install kubectl
- **Windows**: `choco install kubernetes-cli` or download from [Kubernetes releases](https://kubernetes.io/releases/download/)
- **macOS**: `brew install kubectl`
- **Linux**: `curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"`

### 3. Install Helm
- **Windows**: `choco install kubernetes-helm`
- **macOS**: `brew install helm`
- **Linux**: `curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash`

### 4. Install Minikube
- **Windows**: `choco install minikube`
- **macOS**: `brew install minikube`
- **Linux**: `curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/`

### 5. Verify Prerequisites
```bash
# Check Docker
docker --version
docker ps

# Check kubectl
kubectl version --client

# Check Helm
helm version

# Check Minikube
minikube version
```

## Minikube Setup Commands

### 1. Start Minikube Cluster
```bash
# Start with default settings
minikube start

# Or start with specific resources (recommended for this application)
minikube start --cpus=4 --memory=8192 --disk-size=40g

# Enable required addons
minikube addons enable ingress
minikube addons enable metrics-server
```

### 2. Verify Cluster Status
```bash
# Check cluster status
minikube status

# Verify nodes
kubectl get nodes

# Verify system pods
kubectl get pods -n kube-system
```

### 3. Configure kubectl Context
```bash
# Set kubectl context to minikube
kubectl config use-context minikube

# Verify context
kubectl config current-context
```

## Docker Image Building Process

### 1. Build Backend Docker Image
```bash
# Navigate to backend directory
cd backend

# Build the backend image
docker build -t todo-backend:latest .

# Verify the image was created
docker images | grep todo-backend
```

### 2. Build Frontend Docker Image
```bash
# Navigate to frontend directory
cd frontend

# Build the frontend image
docker build -t todo-frontend:latest .

# Verify the image was created
docker images | grep todo-frontend
```

### 3. Tag Images for Minikube
```bash
# Tag images to be compatible with minikube
docker tag todo-backend:latest todo-backend:latest
docker tag todo-frontend:latest todo-frontend:latest
```

## Loading Images to Minikube

### 1. Set Docker Environment to Minikube
```bash
# Configure Docker CLI to point to Minikube's Docker daemon
eval $(minikube docker-env)

# Or on Windows PowerShell:
& minikube docker-env | Invoke-Expression
```

### 2. Build Images in Minikube Context
```bash
# Build backend image in minikube context
cd backend
docker build -t todo-backend:latest .

# Build frontend image in minikube context
cd frontend
docker build -t todo-frontend:latest .
```

### 3. Alternative: Load Images Using Minikube Command
```bash
# Load pre-built images to minikube
minikube image load todo-backend:latest
minikube image load todo-frontend:latest
```

### 4. Verify Images in Minikube
```bash
# List images in minikube
minikube ssh -- docker images
```

## Helm Installation Steps

### 1. Create Namespace
```bash
# Create a dedicated namespace for the application
kubectl create namespace todo-app
```

### 2. Prepare Helm Values
Create a `values.yaml` file with your specific configurations:

```yaml
frontend:
  image:
    repository: todo-frontend
    tag: latest
  replicaCount: 2
  service:
    type: LoadBalancer
    port: 80

backend:
  image:
    repository: todo-backend
    tag: latest
  replicaCount: 2
  service:
    type: ClusterIP
    port: 8000
```

### 3. Install Helm Chart
```bash
# Navigate to the directory containing your Helm chart
cd helm-charts

# Install the chart
helm install todo-app . --namespace todo-app --values values.yaml

# Verify installation
helm list --namespace todo-app
```

### 4. Check Helm Release Status
```bash
# Check the status of the release
helm status todo-app --namespace todo-app

# Get information about the release
helm get all todo-app --namespace todo-app
```

## Deployment Verification Steps

### 1. Check Pod Status
```bash
# Check all pods in the todo-app namespace
kubectl get pods --namespace todo-app

# Check detailed pod information
kubectl describe pods --namespace todo-app

# Check pod logs
kubectl logs -l app=todo-backend --namespace todo-app
kubectl logs -l app=todo-frontend --namespace todo-app
```

### 2. Check Service Status
```bash
# Check services
kubectl get services --namespace todo-app

# Check service endpoints
kubectl get endpoints --namespace todo-app
```

### 3. Check Deployment Status
```bash
# Check deployments
kubectl get deployments --namespace todo-app

# Check deployment details
kubectl describe deployments --namespace todo-app
```

### 4. Verify Application Health
```bash
# Check if health endpoints are responding
kubectl port-forward svc/todo-backend 8000:8000 --namespace todo-app &
curl http://localhost:8000/health

kubectl port-forward svc/todo-frontend 3000:80 --namespace todo-app &
curl http://localhost:3000/health
```

### 5. Check Resource Utilization
```bash
# Check resource usage
kubectl top pods --namespace todo-app

# Check node resources
kubectl top nodes
```

## Port Forwarding Setup

### 1. Forward Backend Service
```bash
# Forward backend service to local port 8000
kubectl port-forward svc/todo-backend 8000:8000 --namespace todo-app
```

### 2. Forward Frontend Service
```bash
# Forward frontend service to local port 3000
kubectl port-forward svc/todo-frontend 3000:80 --namespace todo-app
```

### 3. Access Application
- Frontend: http://localhost:3000
- Backend health check: http://localhost:8000/health

### 4. Alternative Access via Minikube Tunnel
```bash
# Create a tunnel to access LoadBalancer services
minikube tunnel

# Get the external IP of the frontend service
kubectl get svc todo-frontend --namespace todo-app
```

## Scaling Instructions

### 1. Scale Frontend Deployment
```bash
# Scale frontend to 3 replicas
kubectl scale deployment todo-frontend --replicas=3 --namespace todo-app

# Verify scaling
kubectl get pods --namespace todo-app
```

### 2. Scale Backend Deployment
```bash
# Scale backend to 3 replicas
kubectl scale deployment todo-backend --replicas=3 --namespace todo-app

# Verify scaling
kubectl get pods --namespace todo-app
```

### 3. Auto-Scaling with Horizontal Pod Autoscaler (HPA)
```yaml
# Create HPA for backend
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: todo-backend-hpa
  namespace: todo-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: todo-backend
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

```bash
# Apply HPA configuration
kubectl apply -f backend-hpa.yaml

# Check HPA status
kubectl get hpa --namespace todo-app
```

### 4. Check Scaling Events
```bash
# View scaling events
kubectl get events --namespace todo-app --field-selector reason=SuccessfulRescale

# Monitor resource usage
kubectl top pods --namespace todo-app
```

## Troubleshooting Common Issues

### 1. Minikube Won't Start
```bash
# Delete and recreate minikube cluster
minikube delete
minikube start --cpus=4 --memory=8192 --disk-size=40g
```

### 2. Images Not Found in Minikube
```bash
# Ensure Docker environment is set to minikube
eval $(minikube docker-env)  # or minikube docker-env | Invoke-Expression on Windows

# Rebuild images in minikube context
docker build -t todo-backend:latest .
docker build -t todo-frontend:latest .
```

### 3. Services Not Accessible
```bash
# Check if services are running
kubectl get services --namespace todo-app

# Check service configuration
kubectl describe service todo-frontend --namespace todo-app

# Try port forwarding
kubectl port-forward svc/todo-frontend 3000:80 --namespace todo-app
```

### 4. Application Health Checks Failing
```bash
# Check application logs
kubectl logs -l app=todo-backend --namespace todo-app
kubectl logs -l app=todo-frontend --namespace todo-app

# Check if dependencies are available
kubectl exec -it deployment/todo-backend --namespace todo-app -- curl http://todo-database:5432
```

### 5. Insufficient Resources
```bash
# Check resource usage
kubectl top nodes
kubectl describe nodes

# Increase minikube resources
minikube delete
minikube start --cpus=6 --memory=12288 --disk-size=60g
```

## Cleanup and Teardown

### 1. Uninstall Helm Release
```bash
# Uninstall the application
helm uninstall todo-app --namespace todo-app
```

### 2. Delete Namespace
```bash
# Delete the namespace and all resources
kubectl delete namespace todo-app
```

### 3. Stop Minikube
```bash
# Stop the minikube cluster
minikube stop
```

### 4. Delete Minikube Cluster (Optional)
```bash
# Completely delete the cluster
minikube delete
```

## Additional Commands

### 1. Access Minikube Dashboard
```bash
# Start the dashboard
minikube dashboard

# Get the dashboard URL
minikube dashboard --url
```

### 2. SSH into Minikube
```bash
# SSH into the minikube VM
minikube ssh
```

### 3. Get Minikube IP
```bash
# Get the IP address of the minikube cluster
minikube ip
```

This comprehensive guide provides all the necessary steps to deploy the Todo Chatbot application to a local Minikube cluster, including prerequisites, setup, deployment, verification, and troubleshooting.