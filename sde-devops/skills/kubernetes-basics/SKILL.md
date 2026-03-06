---
name: kubernetes-basics
description: "Kubernetes fundamentals for developers: Deployments, Services, ConfigMaps, Secrets, resource limits, health probes, HPA, and kubectl cheatsheet. Use when deploying or debugging an application on Kubernetes."
---

## Kubernetes for Developers

### Context

Kubernetes problem or deployment task: **$ARGUMENTS**

---

### Core Resource Types

```
Pod           — smallest deployable unit; one or more containers
Deployment    — manages replica Pods, handles rolling updates
Service       — stable network endpoint for a set of Pods
ConfigMap     — non-sensitive config as key-value pairs
Secret        — sensitive config (base64-encoded, not encrypted by default)
HPA           — Horizontal Pod Autoscaler — scales replicas based on CPU/memory
Ingress       — HTTP routing rules (URL path → Service)
Namespace     — logical isolation within a cluster
```

---

### Complete Application Manifest

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: production
  labels:
    app: api
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # allow 1 extra pod during update
      maxUnavailable: 0   # never take a pod down before a new one is ready
  template:
    metadata:
      labels:
        app: api
    spec:
      # Security context for the Pod
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001

      containers:
        - name: api
          image: ghcr.io/org/app:sha-abc123  # pin to specific digest in production
          imagePullPolicy: IfNotPresent

          ports:
            - containerPort: 3000
              name: http

          # Environment from ConfigMap and Secrets
          env:
            - name: NODE_ENV
              value: production
            - name: PORT
              value: "3000"
          envFrom:
            - configMapRef:
                name: api-config
            - secretRef:
                name: api-secrets

          # Resource limits (required for HPA to work)
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"    # 100 millicores = 0.1 CPU
            limits:
              memory: "512Mi"
              cpu: "500m"

          # Liveness: is the container alive? Restart if fails.
          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            initialDelaySeconds: 10
            periodSeconds: 30
            failureThreshold: 3

          # Readiness: is the container ready for traffic? Remove from LB if fails.
          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 3

          # Startup probe: give slow-starting apps time to init
          startupProbe:
            httpGet:
              path: /health/live
              port: http
            initialDelaySeconds: 5
            failureThreshold: 30  # allow up to 5*30=150s to start
            periodSeconds: 5

          # Security context for the container
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: [ALL]

          # Mount tmpfs for writable directories
          volumeMounts:
            - name: tmp
              mountPath: /tmp

      volumes:
        - name: tmp
          emptyDir: {}

      # Graceful termination
      terminationGracePeriodSeconds: 30

      # Image pull secret for private registry
      imagePullSecrets:
        - name: ghcr-credentials
```

---

### Service and Ingress

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: production
spec:
  selector:
    app: api           # routes to pods with this label
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
  type: ClusterIP      # internal only; Ingress exposes externally

---
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rate-limit: "100"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
    - hosts: [api.example.com]
      secretName: api-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  name: http
```

---

### ConfigMap and Secrets

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: production
data:
  LOG_LEVEL: info
  ALLOWED_ORIGINS: https://app.example.com
  REDIS_URL: redis://redis-service:6379

---
# k8s/secret.yaml — values must be base64 encoded
# echo -n "mysecret" | base64
apiVersion: v1
kind: Secret
metadata:
  name: api-secrets
  namespace: production
type: Opaque
data:
  JWT_ACCESS_SECRET: <base64>
  JWT_REFRESH_SECRET: <base64>
  MONGODB_URI: <base64>
# Production: use External Secrets Operator to sync from Vault/AWS SSM
```

---

### Horizontal Pod Autoscaler

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70  # scale up when avg CPU > 70%
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60   # wait 60s before scaling up again
    scaleDown:
      stabilizationWindowSeconds: 300  # wait 5min before scaling down (avoid flapping)
```

---

### kubectl Cheatsheet

```bash
# Context and namespace
kubectl config get-contexts
kubectl config use-context my-cluster
kubectl config set-context --current --namespace=production

# Deploy / update
kubectl apply -f k8s/
kubectl rollout status deployment/api
kubectl rollout history deployment/api

# Rollback
kubectl rollout undo deployment/api
kubectl rollout undo deployment/api --to-revision=3

# Debug
kubectl get pods -n production
kubectl describe pod api-abc123 -n production     # events, status
kubectl logs api-abc123 -n production --tail=100  # recent logs
kubectl logs -l app=api -n production --tail=50   # all pods with label
kubectl exec -it api-abc123 -- sh                 # shell into pod

# Resource usage
kubectl top nodes
kubectl top pods -n production

# Scale manually
kubectl scale deployment api --replicas=5

# Port forward (debug without Ingress)
kubectl port-forward deployment/api 3000:3000

# Events (great for debugging scheduling issues)
kubectl get events -n production --sort-by='.lastTimestamp'

# Dry run + diff before applying
kubectl apply -f k8s/ --dry-run=server
kubectl diff -f k8s/
```

---

### Common Deployment Issues

```
CrashLoopBackOff:
  → kubectl describe pod — check Events section
  → kubectl logs <pod> --previous — logs from crashed instance
  → Check liveness probe isn't too aggressive (reduce periodSeconds)
  → Check resource limits aren't too low (OOMKilled)

ImagePullBackOff:
  → Check image name/tag is correct
  → Verify imagePullSecret is configured and valid

Pending (pod not scheduling):
  → kubectl describe pod — look for "Insufficient memory/cpu"
  → Check node capacity: kubectl describe nodes | grep Allocatable
  → Lower resource requests or add nodes

Readiness probe failing:
  → App may be starting slowly — increase initialDelaySeconds
  → Health endpoint may have a bug — test manually: kubectl exec -it <pod> -- wget -qO- localhost:3000/health/ready
```
