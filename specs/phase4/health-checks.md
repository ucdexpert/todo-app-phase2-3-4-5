# Health Checks Specification

## Backend Health Endpoint Specification

### Health Endpoint Implementation
The backend service must implement a health check endpoint at `/health` that returns the status of the application and its dependencies.

#### HTTP Response Format
```json
{
  "status": "healthy",
  "timestamp": "2026-01-03T10:00:00Z",
  "version": "1.0.0",
  "dependencies": {
    "database": {
      "status": "connected",
      "latency": "5ms"
    },
    "openai": {
      "status": "available",
      "latency": "120ms"
    }
  }
}
```

#### Health Check Response Codes
- **200 OK**: Application is healthy and all dependencies are available
- **503 Service Unavailable**: Application is unhealthy or critical dependencies are unavailable

### Health Check Implementation Requirements
1. The health endpoint should respond within 3 seconds
2. It should verify connectivity to the database
3. It should verify connectivity to external services (OpenAI API)
4. It should return a JSON response with the status of all critical dependencies
5. It should include a timestamp of when the check was performed
6. It should include the application version

## Frontend Health Endpoint Specification

### Health Endpoint Implementation
The frontend service must implement a health check endpoint at `/health` that returns the status of the application.

#### HTTP Response Format
```json
{
  "status": "healthy",
  "timestamp": "2026-01-03T10:00:00Z",
  "version": "1.0.0",
  "dependencies": {
    "backend": {
      "status": "reachable",
      "latency": "25ms"
    }
  }
}
```

#### Health Check Response Codes
- **200 OK**: Application is healthy and all dependencies are available
- **503 Service Unavailable**: Application is unhealthy or critical dependencies are unavailable

### Health Check Implementation Requirements
1. The health endpoint should respond within 2 seconds
2. It should verify connectivity to the backend API
3. It should return a JSON response with the status of all critical dependencies
4. It should include a timestamp of when the check was performed
5. It should include the application version

## Liveness Probe Configuration

### Backend Liveness Probe
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
```

#### Configuration Details:
- **initialDelaySeconds**: 30 seconds - Time to wait before first probe after container starts
- **periodSeconds**: 10 seconds - Interval between probes
- **timeoutSeconds**: 5 seconds - Time to wait for probe response
- **failureThreshold**: 3 - Number of consecutive failures before restart

### Frontend Liveness Probe
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
```

#### Configuration Details:
- **initialDelaySeconds**: 30 seconds - Time to wait before first probe after container starts
- **periodSeconds**: 10 seconds - Interval between probes
- **timeoutSeconds**: 5 seconds - Time to wait for probe response
- **failureThreshold**: 3 - Number of consecutive failures before restart

## Readiness Probe Configuration

### Backend Readiness Probe
```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3
```

#### Configuration Details:
- **initialDelaySeconds**: 5 seconds - Time to wait before first probe after container starts
- **periodSeconds**: 5 seconds - Interval between probes
- **timeoutSeconds**: 3 seconds - Time to wait for probe response
- **failureThreshold**: 3 - Number of consecutive failures before marking as unready

### Frontend Readiness Probe
```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3
```

#### Configuration Details:
- **initialDelaySeconds**: 5 seconds - Time to wait before first probe after container starts
- **periodSeconds**: 5 seconds - Interval between probes
- **timeoutSeconds**: 3 seconds - Time to wait for probe response
- **failureThreshold**: 3 - Number of consecutive failures before marking as unready

## Startup Probe Configuration

### Backend Startup Probe
```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 30
```

#### Configuration Details:
- **initialDelaySeconds**: 10 seconds - Time to wait before first probe after container starts
- **periodSeconds**: 5 seconds - Interval between probes
- **timeoutSeconds**: 3 seconds - Time to wait for probe response
- **failureThreshold**: 30 - Number of consecutive failures before restart (allows for 150 seconds total startup time)

### Frontend Startup Probe
```yaml
startupProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 30
```

#### Configuration Details:
- **initialDelaySeconds**: 10 seconds - Time to wait before first probe after container starts
- **periodSeconds**: 5 seconds - Interval between probes
- **timeoutSeconds**: 3 seconds - Time to wait for probe response
- **failureThreshold**: 30 - Number of consecutive failures before restart (allows for 150 seconds total startup time)

## Health Check Best Practices

### 1. Differentiate Between Liveness and Readiness
- **Liveness**: Determines if the application is running properly; restarts if unhealthy
- **Readiness**: Determines if the application is ready to serve traffic; removes from service if not ready

### 2. Implement Appropriate Timeouts
- Set timeouts that account for potential network delays
- Balance between responsiveness and avoiding false failures
- Consider the application's typical response time

### 3. Test Dependencies Appropriately
- For liveness: Check only the application's core functionality
- For readiness: Check dependencies that affect request handling
- Avoid checking external services that could cause cascading failures

### 4. Configure Proper Initial Delays
- Allow sufficient time for applications to initialize
- Consider the application's startup time in different environments
- Account for dependency initialization (database connections, etc.)

### 5. Monitor Health Check Performance
- Track health check response times
- Alert on increasing response times
- Monitor for frequent health check failures

## Health Check Scenarios

### Healthy State
- All health checks return 200 OK
- Response time is within acceptable limits
- All critical dependencies are available

### Unhealthy State - Backend
- Database connection fails
- External API (OpenAI) is unreachable
- Application is experiencing errors

### Unhealthy State - Frontend
- Backend API is unreachable
- Static assets are not loading
- Application is experiencing errors

### Recovery Process
1. Health check fails for the configured failure threshold
2. Kubernetes marks the pod as unhealthy
3. For liveness: Kubernetes restarts the container
4. For readiness: Kubernetes removes the pod from service endpoints
5. New health checks begin after restart or when the issue is resolved

## Troubleshooting Health Checks

### Common Issues and Solutions

#### 1. Health Check Failing After Deployment
- **Cause**: Application not fully initialized
- **Solution**: Increase `initialDelaySeconds` in the probe configuration

#### 2. High CPU Usage Due to Frequent Health Checks
- **Cause**: `periodSeconds` set too low
- **Solution**: Increase `periodSeconds` to reduce check frequency

#### 3. False Positives During Load
- **Cause**: `timeoutSeconds` set too low
- **Solution**: Increase `timeoutSeconds` to account for higher load response times

#### 4. Application Restarting Too Frequently
- **Cause**: `failureThreshold` set too low
- **Solution**: Increase `failureThreshold` to allow for temporary issues

#### 5. Slow Startup Applications
- **Cause**: Startup probe not configured properly
- **Solution**: Configure startup probe with appropriate `failureThreshold` for longer startup times

## Health Check Monitoring

### Metrics to Track
1. Health check response times
2. Health check success/failure rates
3. Pod restart rates due to liveness probe failures
4. Service unavailability due to readiness probe failures

### Alerting Rules
1. Alert when health check failure rate exceeds 5% over 5 minutes
2. Alert when health check response time exceeds 1 second consistently
3. Alert when pods are frequently restarted due to liveness probe failures
4. Alert when services become unavailable due to readiness probe failures

## Implementation Guidelines

### Backend Implementation Example (FastAPI)
```python
from fastapi import FastAPI
from pydantic import BaseModel
from datetime import datetime
import httpx

app = FastAPI()

class HealthStatus(BaseModel):
    status: str
    timestamp: str
    version: str
    dependencies: dict

@app.get("/health", response_model=HealthStatus)
async def health_check():
    dependencies = {}
    
    # Check database connectivity
    try:
        # Implement database connection check
        dependencies["database"] = {"status": "connected", "latency": "5ms"}
    except Exception as e:
        dependencies["database"] = {"status": "error", "error": str(e)}
    
    # Check external services
    try:
        start_time = datetime.now()
        async with httpx.AsyncClient() as client:
            response = await client.get("https://api.openai.com/health")
        latency = f"{(datetime.now() - start_time).total_seconds() * 1000:.0f}ms"
        dependencies["openai"] = {"status": "available", "latency": latency}
    except Exception as e:
        dependencies["openai"] = {"status": "error", "error": str(e)}
    
    # Determine overall status
    overall_status = "healthy"
    for dep_status in dependencies.values():
        if dep_status.get("status") not in ["connected", "available", "reachable"]:
            overall_status = "unhealthy"
            break
    
    return HealthStatus(
        status=overall_status,
        timestamp=datetime.utcnow().isoformat() + "Z",
        version="1.0.0",
        dependencies=dependencies
    )
```

### Frontend Implementation Example (Next.js)
```javascript
// pages/health.js
export default function HealthCheck({ status, timestamp, version, dependencies }) {
  return {
    status,
    timestamp,
    version,
    dependencies
  };
}

export async function getServerSideProps() {
  const timestamp = new Date().toISOString();
  const version = process.env.npm_package_version || "1.0.0";
  
  const dependencies = {};
  
  // Check backend connectivity
  try {
    const start = Date.now();
    const response = await fetch("http://todo-backend:8000/health");
    const latency = `${Date.now() - start}ms`;
    dependencies.backend = {
      status: response.ok ? "reachable" : "unreachable",
      latency
    };
  } catch (error) {
    dependencies.backend = {
      status: "error",
      error: error.message
    };
  }
  
  // Determine overall status
  const overallStatus = dependencies.backend.status === "reachable" 
    ? "healthy" 
    : "unhealthy";
  
  return {
    props: {
      status: overallStatus,
      timestamp,
      version,
      dependencies
    }
  };
}
```

This health check specification ensures that the Todo Chatbot application can be properly monitored and managed in the Kubernetes environment, with appropriate responses to various health states.