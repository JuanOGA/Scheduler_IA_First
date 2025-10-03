# Templates de Configuración Kubernetes

## Objetivo
Proporcionar templates estandarizados para el despliegue de aplicaciones SAAS en Kubernetes, incluyendo configuraciones para desarrollo, staging y producción.

## 1. Namespace y ConfigMaps

### namespace.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: saas-platform
  labels:
    name: saas-platform
    environment: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: saas-platform-staging
  labels:
    name: saas-platform-staging
    environment: staging
---
apiVersion: v1
kind: Namespace
metadata:
  name: saas-platform-dev
  labels:
    name: saas-platform-dev
    environment: development
```

### configmap.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: saas-platform
data:
  # Application configuration
  APP_NAME: "SAAS Platform"
  APP_VERSION: "1.0.0"
  LOG_LEVEL: "info"
  
  # Database configuration
  DB_HOST: "postgres-service"
  DB_PORT: "5432"
  DB_NAME: "saas_db"
  
  # Redis configuration
  REDIS_HOST: "redis-service"
  REDIS_PORT: "6379"
  
  # API configuration
  API_HOST: "0.0.0.0"
  API_PORT: "8000"
  
  # Frontend configuration
  REACT_APP_API_URL: "https://api.saas-platform.com"
  
  # Monitoring
  PROMETHEUS_ENABLED: "true"
  METRICS_PORT: "9090"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: saas-platform
data:
  nginx.conf: |
    events {
        worker_connections 1024;
    }
    
    http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;
        
        # Logging
        log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for"';
        
        access_log /var/log/nginx/access.log main;
        error_log /var/log/nginx/error.log warn;
        
        # Performance
        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        keepalive_timeout 65;
        
        # Gzip compression
        gzip on;
        gzip_vary on;
        gzip_min_length 1024;
        gzip_comp_level 6;
        gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
        
        upstream api_backend {
            server api-service:8000;
        }
        
        server {
            listen 80;
            server_name _;
            
            # Security headers
            add_header X-Frame-Options "SAMEORIGIN" always;
            add_header X-XSS-Protection "1; mode=block" always;
            add_header X-Content-Type-Options "nosniff" always;
            
            # API proxy
            location /api/ {
                proxy_pass http://api_backend/;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
            }
            
            # Frontend
            location / {
                root /usr/share/nginx/html;
                index index.html index.htm;
                try_files $uri $uri/ /index.html;
            }
            
            # Health check
            location /health {
                access_log off;
                return 200 "healthy\n";
                add_header Content-Type text/plain;
            }
        }
    }
```

## 2. Secrets

### secrets.yaml
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: saas-platform
type: Opaque
data:
  # Database credentials (base64 encoded)
  DB_USER: cG9zdGdyZXM=  # postgres
  DB_PASSWORD: cGFzc3dvcmQ=  # password
  
  # JWT secrets
  JWT_SECRET: eW91ci1qd3Qtc2VjcmV0LWtleQ==  # your-jwt-secret-key
  JWT_REFRESH_SECRET: eW91ci1yZWZyZXNoLXNlY3JldA==  # your-refresh-secret
  
  # API keys
  STRIPE_SECRET_KEY: c2tfdGVzdF95b3VyX3N0cmlwZV9rZXk=  # sk_test_your_stripe_key
  SENDGRID_API_KEY: U0cuWW91clNlbmRHcmlkQVBJS2V5  # SG.YourSendGridAPIKey
  
  # OAuth secrets
  GOOGLE_CLIENT_SECRET: eW91ci1nb29nbGUtY2xpZW50LXNlY3JldA==  # your-google-client-secret
  GITHUB_CLIENT_SECRET: eW91ci1naXRodWItY2xpZW50LXNlY3JldA==  # your-github-client-secret
---
apiVersion: v1
kind: Secret
metadata:
  name: registry-secret
  namespace: saas-platform
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: eyJhdXRocyI6eyJ5b3VyLXJlZ2lzdHJ5LmNvbSI6eyJ1c2VybmFtZSI6InVzZXIiLCJwYXNzd29yZCI6InBhc3MiLCJhdXRoIjoiZFhObGNqcHdZWE56In19fQ==
```

## 3. Persistent Volumes

### persistent-volumes.yaml
```yaml
# PostgreSQL Persistent Volume
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/postgres"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: saas-platform
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
# Redis Persistent Volume
apiVersion: v1
kind: PersistentVolume
metadata:
  name: redis-pv
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/redis"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
  namespace: saas-platform
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

## 4. Database Deployments

### postgres-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: saas-platform
  labels:
    app: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_NAME
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: DB_USER
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: DB_PASSWORD
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: saas-platform
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
  type: ClusterIP
```

### redis-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: saas-platform
  labels:
    app: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        command:
        - redis-server
        - --appendonly
        - "yes"
        volumeMounts:
        - name: redis-storage
          mountPath: /data
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: redis-storage
        persistentVolumeClaim:
          claimName: redis-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: saas-platform
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
  type: ClusterIP
```

## 5. Application Deployments

### api-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: saas-platform
  labels:
    app: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      imagePullSecrets:
      - name: registry-secret
      containers:
      - name: api
        image: your-registry.com/saas-platform/api:latest
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_URL
          value: "postgresql://$(DB_USER):$(DB_PASSWORD)@$(DB_HOST):$(DB_PORT)/$(DB_NAME)"
        - name: REDIS_URL
          value: "redis://$(REDIS_HOST):$(REDIS_PORT)"
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_HOST
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_PORT
        - name: DB_NAME
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_NAME
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: DB_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: DB_PASSWORD
        - name: REDIS_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: REDIS_HOST
        - name: REDIS_PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: REDIS_PORT
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: JWT_SECRET
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
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
        startupProbe:
          httpGet:
            path: /health
            port: 8000
          failureThreshold: 30
          periodSeconds: 10
      initContainers:
      - name: migrate
        image: your-registry.com/saas-platform/api:latest
        command: ["python", "manage.py", "migrate"]
        env:
        - name: DATABASE_URL
          value: "postgresql://$(DB_USER):$(DB_PASSWORD)@$(DB_HOST):$(DB_PORT)/$(DB_NAME)"
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_HOST
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_PORT
        - name: DB_NAME
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_NAME
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: DB_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: DB_PASSWORD
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: saas-platform
  labels:
    app: api
spec:
  selector:
    app: api
  ports:
  - port: 8000
    targetPort: 8000
  type: ClusterIP
```

### frontend-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: saas-platform
  labels:
    app: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      imagePullSecrets:
      - name: registry-secret
      containers:
      - name: frontend
        image: your-registry.com/saas-platform/frontend:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
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
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: saas-platform
  labels:
    app: frontend
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

## 6. Ingress Configuration

### ingress.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: saas-platform-ingress
  namespace: saas-platform
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
spec:
  tls:
  - hosts:
    - saas-platform.com
    - api.saas-platform.com
    secretName: saas-platform-tls
  rules:
  - host: saas-platform.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
  - host: api.saas-platform.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8000
```

## 7. Horizontal Pod Autoscaler

### hpa.yaml
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: saas-platform
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
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
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-hpa
  namespace: saas-platform
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

## 8. Network Policies

### network-policies.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
  namespace: saas-platform
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8000
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
  - to: []
    ports:
    - protocol: TCP
      port: 443
    - protocol: TCP
      port: 80
    - protocol: UDP
      port: 53
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-network-policy
  namespace: saas-platform
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 5432
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: redis-network-policy
  namespace: saas-platform
spec:
  podSelector:
    matchLabels:
      app: redis
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 6379
```

## 9. Service Monitor (Prometheus)

### service-monitor.yaml
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: api-service-monitor
  namespace: saas-platform
  labels:
    app: api
spec:
  selector:
    matchLabels:
      app: api
  endpoints:
  - port: metrics
    path: /metrics
    interval: 30s
    scrapeTimeout: 10s
---
apiVersion: v1
kind: Service
metadata:
  name: api-metrics-service
  namespace: saas-platform
  labels:
    app: api
spec:
  selector:
    app: api
  ports:
  - name: metrics
    port: 9090
    targetPort: 9090
  type: ClusterIP
```

## 10. Pod Disruption Budget

### pdb.yaml
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
  namespace: saas-platform
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: api
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: frontend-pdb
  namespace: saas-platform
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: frontend
```

## 11. Jobs y CronJobs

### jobs.yaml
```yaml
# Database backup job
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
  namespace: saas-platform
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: db-backup
            image: postgres:15-alpine
            command:
            - /bin/bash
            - -c
            - |
              pg_dump -h $DB_HOST -U $DB_USER -d $DB_NAME > /backup/backup-$(date +%Y%m%d-%H%M%S).sql
              # Upload to S3 or other storage
            env:
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DB_HOST
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: DB_USER
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: DB_PASSWORD
            - name: DB_NAME
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DB_NAME
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
          restartPolicy: OnFailure
---
# Cleanup old logs job
apiVersion: batch/v1
kind: CronJob
metadata:
  name: log-cleanup
  namespace: saas-platform
spec:
  schedule: "0 1 * * 0"  # Weekly on Sunday at 1 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: log-cleanup
            image: busybox
            command:
            - /bin/sh
            - -c
            - |
              find /var/log -name "*.log" -mtime +7 -delete
              echo "Log cleanup completed"
          restartPolicy: OnFailure
```

## 12. Kustomization

### kustomization.yaml
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: saas-platform

resources:
- namespace.yaml
- configmap.yaml
- secrets.yaml
- persistent-volumes.yaml
- postgres-deployment.yaml
- redis-deployment.yaml
- api-deployment.yaml
- frontend-deployment.yaml
- ingress.yaml
- hpa.yaml
- network-policies.yaml
- service-monitor.yaml
- pdb.yaml
- jobs.yaml

images:
- name: your-registry.com/saas-platform/api
  newTag: v1.0.0
- name: your-registry.com/saas-platform/frontend
  newTag: v1.0.0

patchesStrategicMerge:
- patches/production-patches.yaml

commonLabels:
  version: v1.0.0
  environment: production
```

## 13. Scripts de Despliegue

### deploy.sh
```bash
#!/bin/bash
# Kubernetes deployment script

set -e

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Configuration
NAMESPACE=${1:-saas-platform}
ENVIRONMENT=${2:-production}

echo -e "${GREEN}Deploying to Kubernetes namespace: ${NAMESPACE} (${ENVIRONMENT})${NC}"

# Apply namespace first
echo -e "${YELLOW}Creating namespace...${NC}"
kubectl apply -f namespace.yaml

# Apply secrets and configmaps
echo -e "${YELLOW}Applying secrets and configmaps...${NC}"
kubectl apply -f secrets.yaml
kubectl apply -f configmap.yaml

# Apply persistent volumes
echo -e "${YELLOW}Applying persistent volumes...${NC}"
kubectl apply -f persistent-volumes.yaml

# Deploy databases
echo -e "${YELLOW}Deploying databases...${NC}"
kubectl apply -f postgres-deployment.yaml
kubectl apply -f redis-deployment.yaml

# Wait for databases to be ready
echo -e "${YELLOW}Waiting for databases to be ready...${NC}"
kubectl wait --for=condition=available --timeout=300s deployment/postgres -n ${NAMESPACE}
kubectl wait --for=condition=available --timeout=300s deployment/redis -n ${NAMESPACE}

# Deploy applications
echo -e "${YELLOW}Deploying applications...${NC}"
kubectl apply -f api-deployment.yaml
kubectl apply -f frontend-deployment.yaml

# Wait for applications to be ready
echo -e "${YELLOW}Waiting for applications to be ready...${NC}"
kubectl wait --for=condition=available --timeout=300s deployment/api -n ${NAMESPACE}
kubectl wait --for=condition=available --timeout=300s deployment/frontend -n ${NAMESPACE}

# Apply ingress and other resources
echo -e "${YELLOW}Applying ingress and other resources...${NC}"
kubectl apply -f ingress.yaml
kubectl apply -f hpa.yaml
kubectl apply -f network-policies.yaml
kubectl apply -f service-monitor.yaml
kubectl apply -f pdb.yaml
kubectl apply -f jobs.yaml

echo -e "${GREEN}Deployment completed successfully!${NC}"

# Show status
echo -e "${YELLOW}Current status:${NC}"
kubectl get pods -n ${NAMESPACE}
kubectl get services -n ${NAMESPACE}
kubectl get ingress -n ${NAMESPACE}
```

## Mejores Prácticas

### 1. Seguridad
- Usar Network Policies para segmentar tráfico
- Implementar RBAC apropiado
- Escanear imágenes por vulnerabilidades
- Usar Pod Security Standards

### 2. Observabilidad
- Implementar health checks completos
- Configurar métricas y alertas
- Usar distributed tracing
- Centralizar logs

### 3. Escalabilidad
- Configurar HPA apropiadamente
- Usar Pod Disruption Budgets
- Implementar resource limits
- Planificar para multi-zona

### 4. Mantenimiento
- Automatizar backups
- Implementar rolling updates
- Usar GitOps para despliegues
- Monitorear recursos y costos

Esta documentación proporciona templates completos para desplegar aplicaciones SAAS en Kubernetes de manera segura, escalable y mantenible.