# Helm Charts Specification

## Chart.yaml Structure

The Helm chart for the Todo Chatbot application should follow the standard Helm chart structure with the following Chart.yaml:

```yaml
apiVersion: v2
name: todo-chatbot
description: A Helm chart for deploying the Todo Chatbot application
type: application
version: 0.1.0
appVersion: "1.0.0"
home: https://github.com/your-org/todo-chatbot
sources:
  - https://github.com/your-org/todo-chatbot
maintainers:
  - name: Your Name
    email: your.email@example.com
icon: https://example.com/icon.png  # Optional: URL to an SVG or PNG icon
keywords:
  - todo
  - chatbot
  - ai
  - kubernetes
```

## values.yaml Schema

The values.yaml file should provide configurable parameters for the deployment:

```yaml
# Default values for todo-chatbot

# Frontend configuration
frontend:
  # Frontend image configuration
  image:
    repository: your-registry/todo-frontend
    tag: latest
    pullPolicy: IfNotPresent

  # Frontend service configuration
  service:
    type: ClusterIP
    port: 3000
    targetPort: 3000

  # Frontend resource configuration
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 100m
      memory: 128Mi

  # Frontend replica count
  replicaCount: 2

  # Frontend environment variables
  env:
    NEXT_PUBLIC_API_BASE_URL: "http://todo-backend:8000"
    NEXT_PUBLIC_OPENAI_DOMAIN_KEY: ""
    NEXT_PUBLIC_BETTER_AUTH_URL: ""

  # Frontend health check configuration
  healthCheck:
    path: /health
    initialDelaySeconds: 30
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 3

# Backend configuration
backend:
  # Backend image configuration
  image:
    repository: your-registry/todo-backend
    tag: latest
    pullPolicy: IfNotPresent

  # Backend service configuration
  service:
    type: ClusterIP
    port: 8000
    targetPort: 8000

  # Backend resource configuration
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 100m
      memory: 128Mi

  # Backend replica count
  replicaCount: 2

  # Backend environment variables
  env:
    DATABASE_URL: "postgresql://user:password@postgres:5432/todo"
    OPENAI_API_KEY: ""
    BETTER_AUTH_SECRET: ""
    BETTER_AUTH_URL: ""

  # Backend health check configuration
  healthCheck:
    path: /health
    initialDelaySeconds: 30
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 3

# Database configuration (if using in-cluster database)
database:
  enabled: false  # Set to true if using in-cluster database
  image:
    repository: postgres
    tag: 15-alpine
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 100m
      memory: 128Mi
  env:
    POSTGRES_DB: todo
    POSTGRES_USER: user
    POSTGRES_PASSWORD: password

# Ingress configuration
ingress:
  enabled: false
  className: ""
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  hosts:
    - host: todo.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []
  #  - secretName: todo-tls
  #    hosts:
  #      - todo.local

# Node selector (optional)
nodeSelector: {}

# Tolerations (optional)
tolerations: []

# Affinity (optional)
affinity: {}
```

## Template Files Needed

### 1. templates/frontend-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "todo-chatbot.fullname" . }}-frontend
  labels:
    {{- include "todo-chatbot.labels" . | nindent 4 }}
    app: frontend
spec:
  replicas: {{ .Values.frontend.replicaCount }}
  selector:
    matchLabels:
      {{- include "todo-chatbot.selectorLabels" . | nindent 6 }}
      app: frontend
  template:
    metadata:
      labels:
        {{- include "todo-chatbot.selectorLabels" . | nindent 8 }}
        app: frontend
    spec:
      containers:
        - name: frontend
          image: "{{ .Values.frontend.image.repository }}:{{ .Values.frontend.image.tag }}"
          imagePullPolicy: {{ .Values.frontend.image.pullPolicy }}
          ports:
            - containerPort: 3000
              name: http
          env:
            {{- range $key, $value := .Values.frontend.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
          resources:
            {{- toYaml .Values.frontend.resources | nindent 12 }}
          livenessProbe:
            httpGet:
              path: {{ .Values.frontend.healthCheck.path }}
              port: 3000
            initialDelaySeconds: {{ .Values.frontend.healthCheck.initialDelaySeconds }}
            periodSeconds: {{ .Values.frontend.healthCheck.periodSeconds }}
            timeoutSeconds: {{ .Values.frontend.healthCheck.timeoutSeconds }}
            failureThreshold: {{ .Values.frontend.healthCheck.failureThreshold }}
          readinessProbe:
            httpGet:
              path: {{ .Values.frontend.healthCheck.path }}
              port: 3000
            initialDelaySeconds: {{ .Values.frontend.healthCheck.initialDelaySeconds }}
            periodSeconds: {{ .Values.frontend.healthCheck.periodSeconds }}
            timeoutSeconds: {{ .Values.frontend.healthCheck.timeoutSeconds }}
            failureThreshold: {{ .Values.frontend.healthCheck.failureThreshold }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### 2. templates/backend-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "todo-chatbot.fullname" . }}-backend
  labels:
    {{- include "todo-chatbot.labels" . | nindent 4 }}
    app: backend
spec:
  replicas: {{ .Values.backend.replicaCount }}
  selector:
    matchLabels:
      {{- include "todo-chatbot.selectorLabels" . | nindent 6 }}
      app: backend
  template:
    metadata:
      labels:
        {{- include "todo-chatbot.selectorLabels" . | nindent 8 }}
        app: backend
    spec:
      containers:
        - name: backend
          image: "{{ .Values.backend.image.repository }}:{{ .Values.backend.image.tag }}"
          imagePullPolicy: {{ .Values.backend.image.pullPolicy }}
          ports:
            - containerPort: 8000
              name: http
          env:
            {{- range $key, $value := .Values.backend.env }}
            - name: {{ $key }}
              valueFrom:
                secretKeyRef:
                  name: {{ include "todo-chatbot.fullname" $ }}-backend-secrets
                  key: {{ $key }}
            {{- end }}
          resources:
            {{- toYaml .Values.backend.resources | nindent 12 }}
          livenessProbe:
            httpGet:
              path: {{ .Values.backend.healthCheck.path }}
              port: 8000
            initialDelaySeconds: {{ .Values.backend.healthCheck.initialDelaySeconds }}
            periodSeconds: {{ .Values.backend.healthCheck.periodSeconds }}
            timeoutSeconds: {{ .Values.backend.healthCheck.timeoutSeconds }}
            failureThreshold: {{ .Values.backend.healthCheck.failureThreshold }}
          readinessProbe:
            httpGet:
              path: {{ .Values.backend.healthCheck.path }}
              port: 8000
            initialDelaySeconds: {{ .Values.backend.healthCheck.initialDelaySeconds }}
            periodSeconds: {{ .Values.backend.healthCheck.periodSeconds }}
            timeoutSeconds: {{ .Values.backend.healthCheck.timeoutSeconds }}
            failureThreshold: {{ .Values.backend.healthCheck.failureThreshold }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### 3. templates/frontend-service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "todo-chatbot.fullname" . }}-frontend
  labels:
    {{- include "todo-chatbot.labels" . | nindent 4 }}
    app: frontend
spec:
  type: {{ .Values.frontend.service.type }}
  ports:
    - port: {{ .Values.frontend.service.port }}
      targetPort: {{ .Values.frontend.service.targetPort }}
      protocol: TCP
      name: http
  selector:
    {{- include "todo-chatbot.selectorLabels" . | nindent 4 }}
    app: frontend
```

### 4. templates/backend-service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "todo-chatbot.fullname" . }}-backend
  labels:
    {{- include "todo-chatbot.labels" . | nindent 4 }}
    app: backend
spec:
  type: {{ .Values.backend.service.type }}
  ports:
    - port: {{ .Values.backend.service.port }}
      targetPort: {{ .Values.backend.service.targetPort }}
      protocol: TCP
      name: http
  selector:
    {{- include "todo-chatbot.selectorLabels" . | nindent 4 }}
    app: backend
```

### 5. templates/configmap.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "todo-chatbot.fullname" . }}-config
  labels:
    {{- include "todo-chatbot.labels" . | nindent 4 }}
data:
  # Add any non-sensitive configuration here
  # Example:
  # LOG_LEVEL: "info"
```

### 6. templates/secrets.yaml
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "todo-chatbot.fullname" . }}-backend-secrets
  labels:
    {{- include "todo-chatbot.labels" . | nindent 4 }}
type: Opaque
data:
  # These values should be base64 encoded
  # DATABASE_URL: <base64-encoded-value>
  # OPENAI_API_KEY: <base64-encoded-value>
  # BETTER_AUTH_SECRET: <base64-encoded-value>
  # BETTER_AUTH_URL: <base64-encoded-value>
```

### 7. templates/ingress.yaml (if ingress is enabled)
```yaml
{{- if .Values.ingress.enabled -}}
{{- $fullName := include "todo-chatbot.fullname" . -}}
{{- $svcPort := .Values.frontend.service.port -}}
{{- if and .Values.ingress.className (not (semverCompare ">=1.18-0" .Capabilities.KubeVersion.GitVersion)) }}
  {{- if not (hasKey .Values.ingress.annotations "kubernetes.io/ingress.class") }}
  {{- $_ := set .Values.ingress.annotations "kubernetes.io/ingress.class" .Values.ingress.className}}
  {{- end }}
{{- end }}
{{- if semverCompare ">=1.19-0" .Capabilities.KubeVersion.GitVersion -}}
apiVersion: networking.k8s.io/v1
{{- else if semverCompare ">=1.14-0" .Capabilities.KubeVersion.GitVersion -}}
apiVersion: networking.k8s.io/v1beta1
{{- else -}}
apiVersion: extensions/v1beta1
{{- end }}
kind: Ingress
metadata:
  name: {{ $fullName }}
  labels:
    {{- include "todo-chatbot.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if and .Values.ingress.className (semverCompare ">=1.18-0" .Capabilities.KubeVersion.GitVersion) }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            {{- if and .pathType (semverCompare ">=1.18-0" .Capabilities.KubeVersion.GitVersion) }}
            pathType: {{ .pathType }}
            {{- end }}
            backend:
              {{- if semverCompare ">=1.19-0" $.Capabilities.KubeVersion.GitVersion }}
              service:
                name: {{ $fullName }}-frontend
                port:
                  number: {{ $svcPort }}
              {{- else }}
              serviceName: {{ $fullName }}-frontend
              servicePort: {{ $svcPort }}
              {{- end }}
          {{- end }}
    {{- end }}
{{- end }}
```

## ConfigMap Specification

The ConfigMap should contain non-sensitive configuration values that can be updated without rebuilding the application:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: todo-chatbot-config
  labels:
    app: todo-chatbot
data:
  # Frontend configuration
  NEXT_PUBLIC_API_BASE_URL: "http://todo-backend:8000"
  
  # Backend configuration
  LOG_LEVEL: "INFO"
  DEBUG: "false"
  
  # Database configuration (non-sensitive)
  DB_HOST: "postgres"
  DB_PORT: "5432"
  DB_NAME: "todo"
  
  # Other configuration
  MAX_WORKERS: "4"
```

## Secret Specification

The Secret should contain sensitive information that should not be exposed in plain text:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: todo-chatbot-secrets
  labels:
    app: todo-chatbot
type: Opaque
data:
  # Base64 encoded values
  # DATABASE_URL: <base64-encoded-database-url>
  # OPENAI_API_KEY: <base64-encoded-api-key>
  # BETTER_AUTH_SECRET: <base64-encoded-auth-secret>
  # POSTGRES_PASSWORD: <base64-encoded-password>
```

## Deployment Configuration

### Helm Deployment Commands
```bash
# Install the chart
helm install todo-app ./todo-chatbot --namespace todo-namespace --create-namespace

# Upgrade the chart
helm upgrade todo-app ./todo-chatbot --namespace todo-namespace

# Uninstall the chart
helm uninstall todo-app --namespace todo-namespace
```

### Custom Values File Example
Create a custom-values.yaml file to override default values:

```yaml
frontend:
  image:
    tag: v1.2.3
  replicaCount: 3
  resources:
    limits:
      cpu: 1000m
      memory: 1Gi
    requests:
      cpu: 200m
      memory: 256Mi

backend:
  image:
    tag: v1.2.3
  replicaCount: 3
  resources:
    limits:
      cpu: 1000m
      memory: 1Gi
    requests:
      cpu: 200m
      memory: 256Mi

ingress:
  enabled: true
  hosts:
    - host: todo.example.com
      paths:
        - path: /
          pathType: Prefix
```

## Service Configuration

### Internal Service Communication
Services within the cluster can communicate using the following DNS names:
- Frontend: `todo-chatbot-frontend.todo-namespace.svc.cluster.local`
- Backend: `todo-chatbot-backend.todo-namespace.svc.cluster.local`

### External Access
When using LoadBalancer services or Ingress, the application will be accessible from outside the cluster using the external IP or hostname.

## Best Practices

1. **Security**: Store sensitive information in Secrets, not ConfigMaps
2. **Scalability**: Configure appropriate resource limits and requests
3. **Maintainability**: Use descriptive names and labels for resources
4. **Flexibility**: Provide configurable parameters in values.yaml
5. **Testing**: Test the chart with different configurations before deployment
6. **Documentation**: Document all configurable parameters in the chart README
7. **Versioning**: Use semantic versioning for chart versions
8. **Validation**: Use Helm lint to validate the chart before deployment