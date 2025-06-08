# Kubernetes Toolkit Docker Setup

## Project Structure
```
k8s-toolkit/
├── Dockerfile
├── package.json
├── server.js
├── public/
│   └── index.html
└── k8s-manifests/
    └── deployment.yaml
```

## 1. Create package.json
```json
{
  "name": "k8s-toolkit",
  "version": "1.0.0",
  "description": "Kubernetes cluster management toolkit",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  },
  "engines": {
    "node": ">=16.0.0"
  }
}
```

## 2. Extract server.js
Create `server.js` file with the backend code from the ConfigMap:

```javascript
const express = require('express');
const { exec } = require('child_process');
const path = require('path');
const app = express();
const port = process.env.PORT || 3000;

app.use(express.json());
app.use(express.static('public'));

// API endpoint to execute kubectl commands
app.post('/api/kubectl', (req, res) => {
    const { command, namespace } = req.body;
    
    // Whitelist of allowed commands for security
    const allowedCommands = [
        'get pods', 'get services', 'get deployments', 'get nodes',
        'describe pod', 'describe service', 'describe deployment',
        'logs', 'top pods', 'top nodes', 'get events'
    ];
    
    const isAllowed = allowedCommands.some(allowed => command.startsWith(allowed));
    if (!isAllowed) {
        return res.status(403).json({ error: 'Command not allowed' });
    }

    const kubectlCmd = `kubectl ${command} ${namespace ? `-n ${namespace}` : ''}`;
    
    exec(kubectlCmd, (error, stdout, stderr) => {
        if (error) {
            return res.status(500).json({ error: error.message, stderr });
        }
        res.json({ output: stdout });
    });
});

// Health check endpoint
app.get('/health', (req, res) => {
    res.json({ status: 'healthy', timestamp: new Date().toISOString() });
});

app.listen(port, () => {
    console.log(`K8s Toolkit backend running on port ${port}`);
});
```

## 3. Create Dockerfile
```dockerfile
FROM node:18-alpine

# Install kubectl
ARG KUBECTL_VERSION=v1.28.0
RUN apk add --no-cache curl && \
    curl -LO "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl" && \
    chmod +x kubectl && \
    mv kubectl /usr/local/bin/

# Create app directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy application code
COPY server.js ./
COPY public/ ./public/

# Create non-root user for security
RUN addgroup -g 1001 -S nodejs && \
    adduser -S k8s-toolkit -u 1001 -G nodejs

# Change ownership of app directory
RUN chown -R k8s-toolkit:nodejs /app
USER k8s-toolkit

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

# Start the application
CMD ["npm", "start"]
```

## 4. Create public/index.html
Move the HTML content from the ConfigMap to `public/index.html` (the full HTML content from the ConfigMap).

## 5. Build and Push Docker Image
```bash
# Build the image
docker build -t your-registry/k8s-toolkit:v1.0.0 .

# Tag for latest
docker tag your-registry/k8s-toolkit:v1.0.0 your-registry/k8s-toolkit:latest

# Push to registry
docker push your-registry/k8s-toolkit:v1.0.0
docker push your-registry/k8s-toolkit:latest
```

## 6. Updated Kubernetes Deployment
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: k8s-toolkit
  labels:
    name: k8s-toolkit

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: k8s-toolkit-sa
  namespace: k8s-toolkit

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: k8s-toolkit-reader
rules:
- apiGroups: [""]
  resources: ["pods", "services", "nodes", "events", "configmaps", "secrets", "namespaces"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "daemonsets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["networking.k8s.io"]
  resources: ["networkpolicies", "ingresses"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["metrics.k8s.io"]
  resources: ["pods", "nodes"]
  verbs: ["get", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: k8s-toolkit-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: k8s-toolkit-reader
subjects:
- kind: ServiceAccount
  name: k8s-toolkit-sa
  namespace: k8s-toolkit

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-toolkit
  namespace: k8s-toolkit
  labels:
    app: k8s-toolkit
spec:
  replicas: 2
  selector:
    matchLabels:
      app: k8s-toolkit
  template:
    metadata:
      labels:
        app: k8s-toolkit
    spec:
      serviceAccountName: k8s-toolkit-sa
      containers:
      - name: k8s-toolkit
        image: your-registry/k8s-toolkit:v1.0.0
        ports:
        - containerPort: 3000
          name: http
        env:
        - name: PORT
          value: "3000"
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
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
        securityContext:
          runAsNonRoot: true
          runAsUser: 1001
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL

---
apiVersion: v1
kind: Service
metadata:
  name: k8s-toolkit-service
  namespace: k8s-toolkit
  labels:
    app: k8s-toolkit
spec:
  selector:
    app: k8s-toolkit
  ports:
  - port: 80
    targetPort: 3000
    protocol: TCP
    name: http
  type: ClusterIP

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: k8s-toolkit-ingress
  namespace: k8s-toolkit
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: k8s-toolkit.local  # Change to your domain
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: k8s-toolkit-service
            port:
              number: 80
```

## 7. Deploy to Kubernetes
```bash
# Deploy the application
kubectl apply -f k8s-manifests/deployment.yaml

# Check deployment status
kubectl get pods -n k8s-toolkit

# Check service
kubectl get svc -n k8s-toolkit

# Port forward for testing
kubectl port-forward -n k8s-toolkit service/k8s-toolkit-service 8080:80
```

## Advantages of Docker Image Approach

1. **Proper Dependencies**: All Node.js dependencies are properly managed
2. **Version Control**: Image versioning for releases
3. **Security**: Non-root user, proper health checks
4. **Scalability**: Can easily scale with multiple replicas
5. **Resource Management**: Proper resource limits and requests
6. **Production Ready**: Includes health checks, probes, and security contexts
7. **CI/CD Integration**: Can be built and deployed through pipelines

## Development Workflow
```bash
# Local development
npm install
npm run dev

# Build and test
docker build -t k8s-toolkit:dev .
docker run -p 3000:3000 k8s-toolkit:dev

# Deploy to staging/production
docker build -t your-registry/k8s-toolkit:v1.0.1 .
docker push your-registry/k8s-toolkit:v1.0.1
kubectl set image deployment/k8s-toolkit k8s-toolkit=your-registry/k8s-toolkit:v1.0.1 -n k8s-toolkit
```

This approach is much more maintainable and production-ready compared to using ConfigMaps for application code!
