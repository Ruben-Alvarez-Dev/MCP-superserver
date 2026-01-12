# Kubernetes Deployment Guide

## Overview

This guide covers deploying MCP-SUPERSERVER on Kubernetes clusters.

## Prerequisites

- Kubernetes 1.24+ cluster
- kubectl configured
- Helm 3.x (optional, for Helm charts)
- Persistent storage provisioner
- Ingress controller (optional, for external access)

## Quick Start

### Using Helm (Recommended)

```bash
# Add the Helm repository
helm repo add mcp-superserver https://rubenalvarezdev.github.io/mcp-superserver
helm repo update

# Install
helm install mcp-superserver mcp-superserver/mcp-superserver \
  --namespace mcp-superserver \
  --create-namespace \
  --set neo4j.password=your-secure-password

# Check status
kubectl get pods -n mcp-superserver
```

### Usingkubectl

```bash
# Apply all manifests
kubectl apply -f k8s/

# Check status
kubectl get pods -n mcp-superserver
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Kubernetes Cluster                      │
│                                                               │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                  Namespace: mcp-superserver            │  │
│  │                                                          │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐   │  │
│  │  │ Deployment  │  │ Deployment  │  │ Deployment   │   │  │
│  │  │ mcp-hub     │  │ neo4j       │  │ ollama       │   │  │
│  │  │ (3 replicas)│  │ (1 replica) │  │ (1 replica)  │   │  │
│  │  └─────────────┘  └─────────────┘  └──────────────┘   │  │
│  │                                                          │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐   │  │
│  │  │ StatefulSet │  │ PVC         │  │ PVC          │   │  │
│  │  │ obsidian    │  │ neo4j-data  │  │ obsidian-vault│   │  │
│  │  └─────────────┘  └─────────────┘  └──────────────┘   │  │
│  │                                                          │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐   │  │
│  │  │ Service     │  │ Service     │  │ Service      │   │  │
│  │  │ mcp-hub     │  │ neo4j       │  │ ollama       │   │  │
│  │  └─────────────┘  └─────────────┘  └──────────────┘   │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## Namespace

Create a dedicated namespace:

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mcp-superserver
  labels:
    name: mcp-superserver
```

```bash
kubectl apply -f k8s/namespace.yaml
```

---

## ConfigMap

### Application Configuration

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mcp-hub-config
  namespace: mcp-superserver
data:
  # Node environment
  NODE_ENV: "production"

  # MCP Hub configuration
  MCP_HUB_PORT: "3000"
  MCP_HUB_HOST: "0.0.0.0"

  # Logging
  LOG_LEVEL: "info"
  LOG_FORMAT: "json"
  LOG_RETENTION_DAYS: "30"

  # Service endpoints
  NEO4J_URI: "bolt://neo4j:7687"
  OLLAMA_HOST: "ollama"
  OLLAMA_PORT: "11434"

  # Obsidian configuration
  OBSIDIAN_VAULT_PATH: "/data/obsidian"
  OBSIDIAN_LOGS_FOLDER: "MCP Logs"

  # Protocol Omega
  PROTOCOL_OMEGA_ENABLED: "true"
  PROTOCOL_OMEGA_ENFORCE: "true"
```

```bash
kubectl apply -f k8s/configmap.yaml
```

---

## Secrets

### Create Secrets

```yaml
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mcp-superserver-secrets
  namespace: mcp-superserver
type: Opaque
stringData:
  # Neo4j credentials
  NEO4J_USER: "neo4j"
  NEO4J_PASSWORD: "your-secure-password-here"

  # Optional API key for MCP Hub
  MCP_API_KEY: "your-api-key-here"
```

```bash
# Generate secure password
NEO4J_PASSWORD=$(openssl rand -hex 32)

# Create secret
kubectl create secret generic mcp-superserver-secrets \
  --namespace=mcp-superserver \
  --from-literal=NEO4J_PASSWORD=$NEO4J_PASSWORD \
  --from-literal=MCP_API_KEY=$(openssl rand -hex 16)

# Save password for reference
echo "Neo4j Password: $NEO4J_PASSWORD"
```

---

## Persistent Volume Claims

### Neo4j Data PVC

```yaml
# k8s/pvc-neo4j.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: neo4j-data
  namespace: mcp-superserver
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard # Adjust based on your cluster
```

### Obsidian Vault PVC

```yaml
# k8s/pvc-obsidian.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: obsidian-vault
  namespace: mcp-superserver
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

### Ollama Models PVC

```yaml
# k8s/pvc-ollama.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ollama-models
  namespace: mcp-superserver
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: standard
```

```bash
kubectl apply -f k8s/pvc-neo4j.yaml
kubectl apply -f k8s/pvc-obsidian.yaml
kubectl apply -f k8s/pvc-ollama.yaml
```

---

## Deployments

### Neo4j StatefulSet

```yaml
# k8s/deployment-neo4j.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: neo4j
  namespace: mcp-superserver
spec:
  serviceName: neo4j
  replicas: 1
  selector:
    matchLabels:
      app: neo4j
  template:
    metadata:
      labels:
        app: neo4j
    spec:
      containers:
      - name: neo4j
        image: neo4j:5.15-community
        ports:
        - containerPort: 7474
          name: http
        - containerPort: 7687
          name: bolt
        env:
        - name: NEO4J_AUTH_ENABLED
          value: "true"
        - name: NEO4J_AUTH_USER
          valueFrom:
            secretKeyRef:
              name: mcp-superserver-secrets
              key: NEO4J_USER
        - name: NEO4J_AUTH_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mcp-superserver-secrets
              key: NEO4J_PASSWORD
        - name: NEO4J_dbms_memory_heap_initial__size
          value: "512m"
        - name: NEO4J_dbms_memory_heap_max__size
          value: "2G"
        resources:
          requests:
            memory: "2Gi"
            cpu: "500m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        volumeMounts:
        - name: neo4j-data
          mountPath: /data
        livenessProbe:
          httpGet:
            path: /
            port: 7474
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 7474
          initialDelaySeconds: 10
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: neo4j-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

### Ollama Deployment

```yaml
# k8s/deployment-ollama.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ollama
  namespace: mcp-superserver
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ollama
  template:
    metadata:
      labels:
        app: ollama
    spec:
      containers:
      - name: ollama
        image: ollama/ollama:latest
        ports:
        - containerPort: 11434
          name: http
        resources:
          requests:
            memory: "4Gi"
            cpu: "1000m"
          limits:
            memory: "16Gi"
            cpu: "4000m"
        volumeMounts:
        - name: ollama-models
          mountPath: /root/.ollama
        # Optional: GPU support
        # resources:
        #   limits:
        #     nvidia.com/gpu: 1
```

### MCP Hub Deployment

```yaml
# k8s/deployment-mcp-hub.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-hub
  namespace: mcp-superserver
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mcp-hub
  template:
    metadata:
      labels:
        app: mcp-hub
    spec:
      containers:
      - name: mcp-hub
        image: mcp-superserver/mcp-hub:latest
        ports:
        - containerPort: 3000
          name: http
        envFrom:
        - configMapRef:
            name: mcp-hub-config
        - secretRef:
            name: mcp-superserver-secrets
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        volumeMounts:
        - name: obsidian-vault
          mountPath: /data/obsidian
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
```

```bash
kubectl apply -f k8s/deployment-neo4j.yaml
kubectl apply -f k8s/deployment-ollama.yaml
kubectl apply -f k8s/deployment-mcp-hub.yaml
```

---

## Services

### Neo4j Service

```yaml
# k8s/service-neo4j.yaml
apiVersion: v1
kind: Service
metadata:
  name: neo4j
  namespace: mcp-superserver
spec:
  type: ClusterIP
  ports:
  - port: 7474
    targetPort: 7474
    name: http
  - port: 7687
    targetPort: 7687
    name: bolt
  selector:
    app: neo4j
```

### Ollama Service

```yaml
# k8s/service-ollama.yaml
apiVersion: v1
kind: Service
metadata:
  name: ollama
  namespace: mcp-superserver
spec:
  type: ClusterIP
  ports:
  - port: 11434
    targetPort: 11434
  selector:
    app: ollama
```

### MCP Hub Service

```yaml
# k8s/service-mcp-hub.yaml
apiVersion: v1
kind: Service
metadata:
  name: mcp-hub
  namespace: mcp-superserver
spec:
  type: ClusterIP
  ports:
  - port: 3000
    targetPort: 3000
    name: http
  selector:
    app: mcp-hub
```

```bash
kubectl apply -f k8s/service-neo4j.yaml
kubectl apply -f k8s/service-ollama.yaml
kubectl apply -f k8s/service-mcp-hub.yaml
```

---

## Ingress

### Optional: Expose services externally

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mcp-superserver-ingress
  namespace: mcp-superserver
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - mcp.example.com
    secretName: mcp-superserver-tls
  rules:
  - host: mcp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mcp-hub
            port:
              number: 3000
```

```bash
kubectl apply -f k8s/ingress.yaml
```

---

## Horizontal Pod Autoscaler

### Auto-scale MCP Hub

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: mcp-hub-hpa
  namespace: mcp-superserver
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mcp-hub
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

```bash
kubectl apply -f k8s/hpa.yaml
```

---

## Pod Disruption Budget

### Ensure availability during updates

```yaml
# k8s/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: mcp-hub-pdb
  namespace: mcp-superserver
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: mcp-hub
```

```bash
kubectl apply -f k8s/pdb.yaml
```

---

## Monitoring

### Prometheus ServiceMonitor

```yaml
# k8s/servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mcp-hub
  namespace: mcp-superserver
spec:
  selector:
    matchLabels:
      app: mcp-hub
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
```

---

## Deployment Steps

### Complete deployment process:

```bash
# 1. Create namespace
kubectl apply -f k8s/namespace.yaml

# 2. Create configuration
kubectl apply -f k8s/configmap.yaml

# 3. Create secrets
kubectl create secret generic mcp-superserver-secrets \
  --namespace=mcp-superserver \
  --from-literal=NEO4J_PASSWORD=$(openssl rand -hex 32) \
  --from-literal=MCP_API_KEY=$(openssl rand -hex 16)

# 4. Create PVCs
kubectl apply -f k8s/pvc-neo4j.yaml
kubectl apply -f k8s/pvc-obsidian.yaml
kubectl apply -f k8s/pvc-ollama.yaml

# 5. Deploy Neo4j (wait for ready)
kubectl apply -f k8s/deployment-neo4j.yaml
kubectl wait --for=condition=ready pod -l app=neo4j -n mcp-superserver --timeout=300s

# 6. Deploy Ollama
kubectl apply -f k8s/deployment-ollama.yaml
kubectl wait --for=condition=available deployment/ollama -n mcp-superserver --timeout=300s

# 7. Pull models (optional)
kubectl exec -it -n mcp-superserver deployment/ollama -- ollama pull llama3.3

# 8. Deploy MCP Hub
kubectl apply -f k8s/deployment-mcp-hub.yaml
kubectl wait --for=condition=available deployment/mcp-hub -n mcp-superserver --timeout=180s

# 9. Create services
kubectl apply -f k8s/service-neo4j.yaml
kubectl apply -f k8s/service-ollama.yaml
kubectl apply -f k8s/service-mcp-hub.yaml

# 10. Optional: Ingress
kubectl apply -f k8s/ingress.yaml

# 11. Optional: HPA
kubectl apply -f k8s/hpa.yaml

# 12. Optional: PDB
kubectl apply -f k8s/pdb.yaml

# 13. Verify
kubectl get pods -n mcp-superserver
kubectl get svc -n mcp-superserver
```

---

## Verification

### Check deployment status:

```bash
# Check pods
kubectl get pods -n mcp-superserver

# Check services
kubectl get svc -n mcp-superserver

# Check logs
kubectl logs -n mcp-superserver deployment/mcp-hub -f

# Port forward to test locally
kubectl port-forward -n mcp-superserver svc/mcp-hub 3000:3000

# Test health endpoint
curl http://localhost:3000/health
```

---

## Upgrading

### Rolling update MCP Hub:

```bash
# Build and push new image
docker build -t mcp-superserver/mcp-hub:v1.1.0 -f Docker/mcp-hub/Dockerfile .
docker push mcp-superserver/mcp-hub:v1.1.0

# Update deployment
kubectl set image deployment/mcp-hub \
  mcp-hub=mcp-superserver/mcp-hub:v1.1.0 \
  -n mcp-superserver

# Watch rollout status
kubectl rollout status deployment/mcp-hub -n mcp-superserver

# Rollback if needed
kubectl rollout undo deployment/mcp-hub -n mcp-superserver
```

---

## Scaling

### Manual scaling:

```bash
# Scale MCP Hub to 5 replicas
kubectl scale deployment mcp-hub --replicas=5 -n mcp-superserver

# Scale to 0 (stop)
kubectl scale deployment mcp-hub --replicas=0 -n mcp-superserver
```

### Auto-scaling (see HPA above):

```bash
# Check HPA status
kubectl get hpa -n mcp-superserver

# View HPA events
kubectl describe hpa mcp-hub-hpa -n mcp-superserver
```

---

## Troubleshooting

### Pods not starting:

```bash
# Describe pod
kubectl describe pod -n mcp-superserver <pod-name>

# Check logs
kubectl logs -n mcp-superserver <pod-name>

# Check events
kubectl get events -n mcp-superserver --sort-by='.lastTimestamp'
```

### PVC issues:

```bash
# Check PVC status
kubectl get pvc -n mcp-superserver

# Describe PVC
kubectl describe pvc neo4j-data -n mcp-superserver

# Check storage class
kubectl get storageclass
```

### Service connectivity:

```bash
# Test service from within cluster
kubectl run -it --rm debug --image=busybox --restart=Never -n mcp-superserver -- \
  wget -O- http://mcp-hub:3000/health

# DNS resolution
kubectl run -it --rm debug --image=busybox --restart=Never -n mcp-superserver -- \
  nslookup mcp-hub.mcp-superserver.svc.cluster.local
```

---

## Backup and Restore

### Backup Neo4j data:

```bash
# Create backup job
kubectl create job neo4j-backup --from=cronjob/neo4j-backup -n mcp-superserver

# Or manual backup
kubectl exec -n mcp-superserver statefulset/neo4j -- \
  neo4j-admin dump --database=neo4j --to=/backups/neo4j-backup.dump
```

### Restore Neo4j data:

```bash
# Copy backup to pod
kubectl cp neo4j-backup.dump \
  mcp-superserver/neo4j-0:/backups/neo4j-backup.dump

# Restore
kubectl exec -n mcp-superserver statefulset/neo4j-0 -- \
  neo4j-admin load --from=/backups/neo4j-backup.dump --database=neo4j --force
```

---

## Cleanup

### Remove entire deployment:

```bash
# Delete all resources
kubectl delete namespace mcp-superserver

# Or selective deletion
kubectl delete -f k8s/
```

---

## Production Considerations

### Resource Planning

| Component | Min CPU | Min Memory | Max CPU | Max Memory |
|-----------|---------|------------|---------|------------|
| MCP Hub | 250m | 256Mi | 500m | 512Mi |
| Neo4j | 500m | 2Gi | 2000m | 4Gi |
| Ollama | 1000m | 4Gi | 4000m | 16Gi |

### High Availability

- Use multiple replicas for MCP Hub (3+)
- Consider Neo4j clustering for HA
- Use Pod Disruption Budgets
- Configure proper anti-affinity rules

### Security

- Use NetworkPolicies to restrict traffic
- Enable PodSecurityStandards
- Rotate secrets regularly
- Use TLS for external traffic
- Implement RBAC

### Monitoring

- Deploy Prometheus + Grafana
- Set up alerts for pod failures
- Monitor resource usage
- Track application metrics

---

## Helm Chart

For easier deployment, a Helm chart is available:

```bash
# Install from repository
helm install mcp-superserver mcp-superserver/mcp-superserver \
  --namespace mcp-superserver \
  --create-namespace \
  --values values.yaml

# Custom values.yaml
neo4j:
  password: your-secure-password
  resources:
    requests:
      memory: 4Gi
      cpu: 1000m

ollama:
  models:
    - llama3.3
    - qwq
  resources:
    limits:
      nvidia.com/gpu: 1

mcpHub:
  replicas: 3
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
```

---

## Next Steps

1. Configure your CLI tools to connect to the Kubernetes deployment
2. Set up monitoring and alerting
3. Configure backup schedules
4. Review security best practices
5. Document your deployment procedures
