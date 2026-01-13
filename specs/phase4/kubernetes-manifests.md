# Kubernetes Manifests Specification

## Backend Deployment Spec (2 replicas)

### Deployment Configuration
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-backend
  namespace: todo-app
  labels:
    app: todo-backend
    version: v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: todo-backend
  template:
    metadata:
      labels:
        app: todo-backend
        version: v1
    spec:
      containers:
      - name: backend
        image: todo-backend:latest
        ports:
        - containerPort: 8000
          name: http
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: todo-backend-secrets
              key: database-url
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: todo-backend-secrets
              key: openai-api-key
        - name: BETTER_AUTH_SECRET
          valueFrom:
            secretKeyRef:
              name: todo-backend-secrets
              key: auth-secret
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        startupProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 30
      restartPolicy: Always
```

### Backend Service Configuration
```yaml
apiVersion: v1
kind: Service
metadata:
  name: todo-backend
  namespace: todo-app
  labels:
    app: todo-backend
spec:
  selector:
    app: todo-backend
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 8000
      name: http
  type: ClusterIP
```

## Frontend Deployment Spec (2 replicas)

### Deployment Configuration
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-frontend
  namespace: todo-app
  labels:
    app: todo-frontend
    version: v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: todo-frontend
  template:
    metadata:
      labels:
        app: todo-frontend
        version: v1
    spec:
      containers:
      - name: frontend
        image: todo-frontend:latest
        ports:
        - containerPort: 3000
          name: http
        env:
        - name: NEXT_PUBLIC_API_BASE_URL
          value: "http://todo-backend:8000"
        - name: NEXT_PUBLIC_OPENAI_DOMAIN_KEY
          valueFrom:
            secretKeyRef:
              name: todo-frontend-secrets
              key: openai-domain-key
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        startupProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 30
      restartPolicy: Always
```

### Frontend Service Configuration
```yaml
apiVersion: v1
kind: Service
metadata:
  name: todo-frontend
  namespace: todo-app
  labels:
    app: todo-frontend
spec:
  selector:
    app: todo-frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
      name: http
  type: LoadBalancer  # For external access in development
```

## Service Definitions

### Backend Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: todo-backend
  namespace: todo-app
  labels:
    app: todo-backend
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  selector:
    app: todo-backend
  ports:
    - name: http
      protocol: TCP
      port: 8000
      targetPort: 8000
  type: ClusterIP  # Internal service only
  sessionAffinity: None
```

### Frontend Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: todo-frontend
  namespace: todo-app
  labels:
    app: todo-frontend
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  selector:
    app: todo-frontend
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 3000
    - name: https
      protocol: TCP
      port: 443
      targetPort: 3000
  type: LoadBalancer  # External access
  sessionAffinity: None
```

### Database Service (if using in-cluster database)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: todo-database
  namespace: todo-app
  labels:
    app: todo-database
spec:
  selector:
    app: todo-database
  ports:
    - name: postgres
      protocol: TCP
      port: 5432
      targetPort: 5432
  type: ClusterIP
```

## Resource Limits and Requests

### Backend Resource Configuration
```yaml
resources:
  requests:
    memory: "256Mi"    # Minimum memory required
    cpu: "250m"        # Minimum CPU required (0.25 core)
  limits:
    memory: "512Mi"    # Maximum memory allowed
    cpu: "500m"        # Maximum CPU allowed (0.5 core)
```

### Frontend Resource Configuration
```yaml
resources:
  requests:
    memory: "128Mi"    # Minimum memory required
    cpu: "100m"        # Minimum CPU required (0.1 core)
  limits:
    memory: "256Mi"    # Maximum memory allowed
    cpu: "200m"        # Maximum CPU allowed (0.2 core)
```

### Database Resource Configuration (if applicable)
```yaml
resources:
  requests:
    memory: "512Mi"    # Minimum memory required
    cpu: "500m"        # Minimum CPU required (0.5 core)
  limits:
    memory: "1Gi"      # Maximum memory allowed
    cpu: "1000m"       # Maximum CPU allowed (1 core)
```

## Labels and Selectors

### Standard Labels
All Kubernetes resources should include these standard labels:

```yaml
labels:
  app: todo-app                    # Name of the application
  component: <frontend|backend>    # Component name
  version: v1                      # Version of the component
  environment: development         # Environment (development/staging/production)
  tier: <frontend|backend>         # Tier of the application
  team: todo-team                  # Team responsible for the component
```

### Selector Examples
Selectors are used to match pods with services and deployments:

```yaml
selector:
  matchLabels:
    app: todo-backend
    version: v1
```

### Label Selectors for Different Resources

#### Deployment Selector
```yaml
spec:
  selector:
    matchLabels:
      app: todo-backend
  template:
    metadata:
      labels:
        app: todo-backend
        version: v1
```

#### Service Selector
```yaml
spec:
  selector:
    app: todo-backend  # Must match pod labels
  ports:
  - port: 8000
    targetPort: 8000
```

## Additional Kubernetes Resources

### ConfigMap for Non-Sensitive Configuration
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: todo-app-config
  namespace: todo-app
data:
  LOG_LEVEL: "info"
  DEBUG: "false"
  NEXT_PUBLIC_API_BASE_URL: "http://todo-backend:8000"
```

### Secret for Sensitive Information
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: todo-app-secrets
  namespace: todo-app
type: Opaque
data:
  # All values must be base64 encoded
  database-url: <base64-encoded-url>
  openai-api-key: <base64-encoded-key>
  auth-secret: <base64-encoded-secret>
```

### Network Policy (Optional)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: todo-app-network-policy
  namespace: todo-app
spec:
  podSelector:
    matchLabels:
      app: todo-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: todo-frontend
    ports:
    - protocol: TCP
      port: 8000
  egress:
  - to: []
    ports:
    - protocol: TCP
      port: 5432  # Database
    - protocol: TCP
      port: 443   # External APIs
```

### Horizontal Pod Autoscaler (Optional)
```yaml
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

## Deployment Strategy

### Rolling Update Configuration
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max additional pods during update
      maxUnavailable: 1  # Max unavailable pods during update
  replicas: 2
```

### Health Check Integration
All deployments include health checks to ensure proper operation:
- Liveness probes to restart unhealthy containers
- Readiness probes to remove unhealthy containers from service
- Startup probes for slow-starting applications

## Namespace Configuration

### Namespace Definition
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: todo-app
  labels:
    name: todo-app
    environment: development
```

This specification provides a complete set of Kubernetes manifests for deploying the Todo Chatbot application with proper resource management, health checks, and service discovery.