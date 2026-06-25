# Deployment Guide

Panduan lengkap untuk deploy Bee Group Platform ke production.

---

## 🚀 Deployment Overview

### Deployment Architecture

```
┌──────────────────────────────────────────┐
│         Client Browsers                  │
│  (React Frontend - SPA)                  │
└──────────────────┬───────────────────────┘
                   │
         ┌─────────┴──────────┐
         │                    │
    ┌────▼─────┐      ┌──────▼─────┐
    │   CDN    │      │   API GW   │
    │(Cloudflare)     │(Nginx)     │
    └────┬─────┘      └──────┬─────┘
         │                   │
    ┌────▼───────────────────▼────┐
    │  Load Balancer               │
    │  (AWS ELB / Nginx)           │
    └────┬───────────────────┬────┘
         │                   │
    ┌────▼────┐         ┌────▼────┐
    │Backend 1│         │Backend 2│
    │ (Node)  │         │ (Node)  │
    └────┬────┘         └────┬────┘
         │                   │
    ┌────▼───────────────────▼────┐
    │  Database (PostgreSQL)       │
    │  + Backup/Replication       │
    └─────────────────────────────┘
```

---

## 📋 Pre-Deployment Checklist

### Development to Production

- [ ] All tests passing locally
- [ ] Code review completed
- [ ] No console.log/debug statements
- [ ] Environment variables configured
- [ ] Database migrations tested
- [ ] Security audit passed
- [ ] Performance testing completed
- [ ] Backup strategy ready
- [ ] Monitoring configured
- [ ] Documentation updated

### Security Checklist

- [ ] HTTPS/SSL configured
- [ ] CORS properly set
- [ ] Rate limiting enabled
- [ ] Database encrypted
- [ ] Sensitive data not in logs
- [ ] No hardcoded credentials
- [ ] API keys rotated
- [ ] Firewall rules configured

---

## 🐳 Docker Deployment

### Build Docker Images

```bash
# Build backend image
docker build -t bee-group-api:1.0.0 ./backend

# Build frontend image
docker build -t bee-group-web:1.0.0 ./frontend

# Tag for registry
docker tag bee-group-api:1.0.0 registry.example.com/bee-group-api:1.0.0
docker tag bee-group-web:1.0.0 registry.example.com/bee-group-web:1.0.0

# Push to registry
docker push registry.example.com/bee-group-api:1.0.0
docker push registry.example.com/bee-group-web:1.0.0
```

### Production Docker Compose

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    container_name: bee-group-db
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./backups:/backups
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: always
    networks:
      - bee-group-network

  redis:
    image: redis:7-alpine
    container_name: bee-group-cache
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: always
    networks:
      - bee-group-network
    command: redis-server --appendonly yes

  backend:
    image: registry.example.com/bee-group-api:1.0.0
    container_name: bee-group-api
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://${DB_USER}:${DB_PASSWORD}@postgres:5432/${DB_NAME}
      REDIS_URL: redis://redis:6379
      JWT_SECRET: ${JWT_SECRET}
      API_PORT: 3000
      CORS_ORIGIN: ${CORS_ORIGIN}
    ports:
      - "3000:3000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: always
    networks:
      - bee-group-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  frontend:
    image: registry.example.com/bee-group-web:1.0.0
    container_name: bee-group-web
    environment:
      VITE_API_URL: ${VITE_API_URL}
    ports:
      - "80:5173"
    depends_on:
      - backend
    restart: always
    networks:
      - bee-group-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5173"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local

networks:
  bee-group-network:
    driver: bridge
```

---

## ☸️ Kubernetes Deployment

### Prerequisites
- Kubernetes cluster (1.24+)
- kubectl configured
- Helm (optional)
- Docker registry

### Namespace & ConfigMap

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: bee-group

---
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: bee-group-config
  namespace: bee-group
data:
  NODE_ENV: production
  API_PORT: "3000"
  CORS_ORIGIN: "https://app.beegroup.com"
  VITE_API_URL: "https://api.beegroup.com"
```

### Secrets

```yaml
# secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: bee-group-secrets
  namespace: bee-group
type: Opaque
data:
  DATABASE_URL: cG9zdGdyZXM6Ly91c2VyOnBhc3NAaG9zdC9kYm5hbWU=  # base64 encoded
  JWT_SECRET: eW91ci1zZWNyZXQta2V5LWhlcmU=
  DB_PASSWORD: cGFzc3dvcmQxMjM=
```

### PostgreSQL StatefulSet

```yaml
# postgres-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: bee-group
spec:
  serviceName: postgres
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
        image: postgres:16-alpine
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_DB
          value: bee_group
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: bee-group-secrets
              key: DB_USER
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: bee-group-secrets
              key: DB_PASSWORD
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - pg_isready -U postgres
          initialDelaySeconds: 30
          periodSeconds: 10
      volumeClaimTemplates:
      - metadata:
          name: postgres-storage
        spec:
          accessModes: [ "ReadWriteOnce" ]
          resources:
            requests:
              storage: 50Gi
```

### Backend Deployment

```yaml
# backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bee-group-api
  namespace: bee-group
spec:
  replicas: 3
  selector:
    matchLabels:
      app: bee-group-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: bee-group-api
    spec:
      containers:
      - name: api
        image: registry.example.com/bee-group-api:1.0.0
        imagePullPolicy: Always
        ports:
        - containerPort: 3000
          name: http
        env:
        - name: NODE_ENV
          valueFrom:
            configMapKeyRef:
              name: bee-group-config
              key: NODE_ENV
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: bee-group-secrets
              key: DATABASE_URL
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: bee-group-secrets
              key: JWT_SECRET
        livenessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

### Backend Service

```yaml
# backend-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: bee-group-api
  namespace: bee-group
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 3000
    protocol: TCP
    name: http
  selector:
    app: bee-group-api
```

### Frontend Deployment

```yaml
# frontend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bee-group-web
  namespace: bee-group
spec:
  replicas: 3
  selector:
    matchLabels:
      app: bee-group-web
  template:
    metadata:
      labels:
        app: bee-group-web
    spec:
      containers:
      - name: web
        image: registry.example.com/bee-group-web:1.0.0
        imagePullPolicy: Always
        ports:
        - containerPort: 5173
          name: http
        env:
        - name: VITE_API_URL
          valueFrom:
            configMapKeyRef:
              name: bee-group-config
              key: VITE_API_URL
        livenessProbe:
          httpGet:
            path: /
            port: 5173
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 5173
          initialDelaySeconds: 10
          periodSeconds: 5
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "250m"
```

### Deploy to Kubernetes

```bash
# Create namespace
kubectl apply -f namespace.yaml

# Create secrets and configmaps
kubectl apply -f configmap.yaml
kubectl apply -f secrets.yaml

# Deploy database
kubectl apply -f postgres-statefulset.yaml

# Deploy backend
kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml

# Deploy frontend
kubectl apply -f frontend-deployment.yaml

# Check deployment status
kubectl get deployments -n bee-group
kubectl get pods -n bee-group
kubectl get services -n bee-group
```

---

## 🌍 Cloud Deployment (AWS)

### EC2 + RDS + S3

```bash
# 1. Launch EC2 instances
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.medium \
  --key-name bee-group-key \
  --security-groups bee-group-sg \
  --user-data file://user-data.sh

# 2. Create RDS PostgreSQL
aws rds create-db-instance \
  --db-instance-identifier bee-group-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --master-username admin \
  --allocated-storage 100 \
  --backup-retention-period 30 \
  --multi-az \
  --storage-encrypted

# 3. Create S3 bucket for backups
aws s3 mb s3://bee-group-backups-prod \
  --region ap-southeast-1

# 4. Create load balancer
aws elb create-load-balancer \
  --load-balancer-name bee-group-lb \
  --listeners Protocol=HTTP,LoadBalancerPort=80,InstancePort=3000
```

### AWS ECS Deployment

```bash
# Create ECS cluster
aws ecs create-cluster --cluster-name bee-group

# Create task definition
aws ecs register-task-definition \
  --cli-input-json file://task-definition.json

# Create service
aws ecs create-service \
  --cluster bee-group \
  --service-name bee-group-api \
  --task-definition bee-group-api:1 \
  --desired-count 3
```

---

## 🔄 CI/CD Pipeline

### GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '20'
      
      - name: Backend Tests
        run: |
          cd backend
          npm ci
          npm run test:ci
      
      - name: Frontend Tests
        run: |
          cd frontend
          npm ci
          npm run test

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Login to Docker Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ secrets.DOCKER_REGISTRY }}
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      
      - name: Build & Push Backend
        uses: docker/build-push-action@v4
        with:
          context: ./backend
          push: true
          tags: |
            ${{ secrets.DOCKER_REGISTRY }}/bee-group-api:${{ github.sha }}
            ${{ secrets.DOCKER_REGISTRY }}/bee-group-api:latest
      
      - name: Build & Push Frontend
        uses: docker/build-push-action@v4
        with:
          context: ./frontend
          push: true
          tags: |
            ${{ secrets.DOCKER_REGISTRY }}/bee-group-web:${{ github.sha }}
            ${{ secrets.DOCKER_REGISTRY }}/bee-group-web:latest

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to Production
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.DEPLOY_KEY }}" > ~/.ssh/deploy_key
          chmod 600 ~/.ssh/deploy_key
          ssh-keyscan -H ${{ secrets.DEPLOY_HOST }} >> ~/.ssh/known_hosts
          
          ssh -i ~/.ssh/deploy_key deployer@${{ secrets.DEPLOY_HOST }} << 'EOF'
            cd /opt/bee-group
            docker-compose pull
            docker-compose up -d
            docker-compose exec -T backend npm run db:migrate
          EOF
      
      - name: Verify Deployment
        run: |
          curl -f https://api.beegroup.com/api/health || exit 1
```

---

## 📊 Monitoring & Logging

### Application Monitoring

```bash
# Install Prometheus & Grafana
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus

docker run -d \
  --name grafana \
  -p 3000:3000 \
  grafana/grafana
```

### Centralized Logging

```bash
# Install ELK Stack
docker run -d \
  --name elasticsearch \
  -p 9200:9200 \
  -e "discovery.type=single-node" \
  docker.elastic.co/elasticsearch/elasticsearch:8.0.0

docker run -d \
  --name kibana \
  -p 5601:5601 \
  -e "ELASTICSEARCH_HOSTS=http://elasticsearch:9200" \
  docker.elastic.co/kibana/kibana:8.0.0
```

### Application Metrics

```javascript
// backend/src/utils/monitoring.js
import prometheus from 'prom-client';

// Create metrics
const httpRequestDuration = new prometheus.Histogram({
  name: 'http_request_duration_ms',
  help: 'Duration of HTTP requests in ms',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.1, 5, 15, 50, 100, 500]
});

const dbQueryDuration = new prometheus.Histogram({
  name: 'db_query_duration_ms',
  help: 'Duration of database queries in ms',
  labelNames: ['query_type'],
  buckets: [0.1, 5, 15, 50, 100, 500]
});

// Middleware untuk tracking
export const metricsMiddleware = (req, res, next) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - start;
    httpRequestDuration
      .labels(req.method, req.route?.path || req.path, res.statusCode)
      .observe(duration);
  });
  
  next();
};

// Expose metrics endpoint
app.get('/metrics', (req, res) => {
  res.set('Content-Type', prometheus.register.contentType);
  res.end(prometheus.register.metrics());
});
```

---

## 🔙 Backup & Recovery

### Database Backup Strategy

```bash
#!/bin/bash
# backup.sh - Daily backup script

BACKUP_DIR="/backups/bee-group"
DB_NAME="bee_group"
DB_USER="postgres"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Create backup directory
mkdir -p $BACKUP_DIR

# Full backup
pg_dump -U $DB_USER $DB_NAME | gzip > $BACKUP_DIR/full_$TIMESTAMP.sql.gz

# Upload to S3
aws s3 cp $BACKUP_DIR/full_$TIMESTAMP.sql.gz s3://bee-group-backups/

# Clean old backups (keep last 30 days)
find $BACKUP_DIR -type f -mtime +30 -delete

# Alert on backup completion
echo "Backup completed: full_$TIMESTAMP.sql.gz" | mail -s "Bee Group Backup" ops@beegroup.com
```

### Restore from Backup

```bash
#!/bin/bash
# restore.sh - Restore from backup

BACKUP_FILE=$1
DB_NAME="bee_group"
DB_USER="postgres"

if [ -z "$BACKUP_FILE" ]; then
  echo "Usage: $0 <backup_file>"
  exit 1
fi

# Create new database
createdb -U $DB_USER "${DB_NAME}_restore"

# Restore backup
gunzip -c "$BACKUP_FILE" | psql -U $DB_USER -d "${DB_NAME}_restore"

echo "Restored to: ${DB_NAME}_restore"
```

---

## 🚨 Rollback Strategy

### Blue-Green Deployment

```yaml
# blue-deployment.yaml (Current production)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bee-group-api-blue
spec:
  replicas: 3
  # ... rest of spec

---
# green-deployment.yaml (New version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bee-group-api-green
spec:
  replicas: 3
  # ... rest of spec

---
# service.yaml (Points to blue/green)
apiVersion: v1
kind: Service
metadata:
  name: bee-group-api
spec:
  selector:
    app: bee-group-api-blue  # Switch to green after testing
```

### Quick Rollback

```bash
# If deployment fails, immediately rollback
kubectl rollout undo deployment/bee-group-api -n bee-group

# Or rollback to specific revision
kubectl rollout history deployment/bee-group-api -n bee-group
kubectl rollout undo deployment/bee-group-api --to-revision=2 -n bee-group
```

---

## 📋 Post-Deployment

### Verification Checklist

- [ ] Application is running
- [ ] API endpoints responding
- [ ] Database migrations successful
- [ ] Frontend assets loaded
- [ ] SSL certificate valid
- [ ] Monitoring alerts working
- [ ] Logs being collected
- [ ] Backups completed
- [ ] Performance acceptable
- [ ] No errors in logs

### Health Checks

```bash
# Test API health
curl https://api.beegroup.com/api/health

# Test frontend
curl https://app.beegroup.com

# Test database connection
curl https://api.beegroup.com/api/companies

# Monitor logs
kubectl logs -f deployment/bee-group-api -n bee-group
```

---

## 🆘 Troubleshooting

### Common Issues

#### Pod won't start
```bash
kubectl describe pod bee-group-api-xxx -n bee-group
kubectl logs bee-group-api-xxx -n bee-group
```

#### Database connection failed
```bash
# Check if database is accessible
pg_isready -h postgres-service -U postgres

# Check connection string
echo $DATABASE_URL
```

#### Out of memory
```bash
# Check resource usage
kubectl top nodes
kubectl top pods -n bee-group

# Increase limits
kubectl set resources deployment bee-group-api \
  --limits=memory=1Gi,cpu=1000m \
  -n bee-group
```

---

**Last Updated**: June 25, 2024
**Version**: 1.0.0
