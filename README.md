# Cloud-Native DevOps Transformation — Manufacturing Use Case

![DevOps](https://img.shields.io/badge/Domain-Cloud%20Native%20DevOps-blue)
![Docker](https://img.shields.io/badge/Tool-Docker-blue)
![Kubernetes](https://img.shields.io/badge/Tool-Kubernetes-blue)
![Terraform](https://img.shields.io/badge/IaC-Terraform-purple)
![GitHub Actions](https://img.shields.io/badge/CI%2FCD-GitHub%20Actions-black)
![Jenkins](https://img.shields.io/badge/CI%2FCD-Jenkins-red)
![Prometheus](https://img.shields.io/badge/Monitoring-Prometheus-orange)
![Grafana](https://img.shields.io/badge/Monitoring-Grafana-orange)

---

## 📋 Project Overview

This project documents a complete cloud-native DevOps transformation for a manufacturing enterprise — migrating from legacy monolithic applications to containerised microservices using Docker and Kubernetes with full CI/CD automation, infrastructure as code, comprehensive monitoring, and FinOps cost optimisation.

Built to demonstrate operational readiness for:
- Cloud & DevOps Engineer roles
- Cloud Infrastructure Engineer roles
- Platform Engineer roles
- Site Reliability Engineer roles

---

## 🏭 Manufacturing Use Case Context

This transformation addresses common manufacturing IT challenges:

| Challenge | Solution |
|---|---|
| Legacy monolithic ERP | Containerised microservices |
| Manual deployments | Automated CI/CD pipelines |
| No deployment consistency | Infrastructure as Code |
| Limited visibility | Prometheus & Grafana monitoring |
| High infrastructure costs | FinOps optimisation |
| Slow release cycles | GitHub Actions automation |

---

## Repository Structure

---

## Architecture Overview
---

## Docker Configuration

### Dockerfile — Manufacturing ERP Service

```dockerfile
# Multi-stage build for production optimization
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

# Production stage
FROM node:18-alpine AS production

# Security — run as non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

WORKDIR /app

# Copy built artifacts from builder
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:appgroup /app/package.json ./

USER appuser

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "dist/server.js"]
```

### Docker Compose — Local Development

```yaml
version: '3.8'

services:
  erp-app:
    build:
      context: .
      target: production
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - DB_HOST=postgres
      - REDIS_HOST=redis
    depends_on:
      - postgres
      - redis
    networks:
      - manufacturing-net

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: manufacturing
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - manufacturing-net

  redis:
    image: redis:7-alpine
    networks:
      - manufacturing-net

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    networks:
      - manufacturing-net

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    networks:
      - manufacturing-net

volumes:
  postgres-data:
  grafana-data:

networks:
  manufacturing-net:
    driver: bridge
```

---

## Kubernetes Configuration

### Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: erp-app
  namespace: production
  labels:
    app: erp-app
    version: v1.0.0
    environment: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: erp-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: erp-app
        version: v1.0.0
    spec:
      serviceAccountName: erp-app-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001
      containers:
        - name: erp-app
          image: ${ECR_REGISTRY}/erp-app:${IMAGE_TAG}
          ports:
            - containerPort: 3000
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
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
          env:
            - name: NODE_ENV
              value: "production"
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: host
```

### Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: erp-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: erp-app
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
```

---

## CI/CD Pipeline — GitHub Actions

```yaml
name: Manufacturing DevOps Pipeline

on:
  push:
    branches: [main, staging]
  pull_request:
    branches: [main]

env:
  ECR_REGISTRY: ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.us-east-1.amazonaws.com
  IMAGE_NAME: erp-app

jobs:
  build-and-test:
    name: Build and Test
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run unit tests
        run: npm test

      - name: Run integration tests
        run: npm run test:integration

  security-scan:
    name: Security Scanning
    runs-on: ubuntu-latest
    needs: build-and-test
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Build Docker image for scanning
        run: docker build -t $IMAGE_NAME:scan .

      - name: Run Trivy vulnerability scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.IMAGE_NAME }}:scan
          format: sarif
          exit-code: '1'
          severity: CRITICAL,HIGH

  build-and-push:
    name: Build and Push to ECR
    runs-on: ubuntu-latest
    needs: security-scan
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Build and push Docker image
        id: meta
        run: |
          IMAGE_TAG=${GITHUB_SHA::8}
          docker build -t $ECR_REGISTRY/$IMAGE_NAME:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$IMAGE_NAME:$IMAGE_TAG
          echo "tags=$ECR_REGISTRY/$IMAGE_NAME:$IMAGE_TAG" >> $GITHUB_OUTPUT

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build-and-push
    environment: staging
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name manufacturing-cluster --region us-east-1

      - name: Deploy to staging
        run: |
          kubectl set image deployment/erp-app \
            erp-app=${{ needs.build-and-push.outputs.image-tag }} \
            -n staging
          kubectl rollout status deployment/erp-app -n staging

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: production
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name manufacturing-cluster --region us-east-1

      - name: Deploy to production
        run: |
          kubectl set image deployment/erp-app \
            erp-app=${{ needs.build-and-push.outputs.image-tag }} \
            -n production
          kubectl rollout status deployment/erp-app -n production --timeout=5m
```

---

## Prometheus Monitoring Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - "manufacturing_alerts.yml"

scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true

  - job_name: 'erp-app'
    static_configs:
      - targets: ['erp-app-service:3000']
    metrics_path: '/metrics'

  - job_name: 'node-exporter'
    kubernetes_sd_configs:
      - role: node

  - job_name: 'kube-state-metrics'
    static_configs:
      - targets: ['kube-state-metrics:8080']
```

### Manufacturing Alert Rules

```yaml
# manufacturing_alerts.yml
groups:
  - name: manufacturing_alerts
    rules:
      - alert: ERP_ServiceDown
        expr: up{job="erp-app"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "ERP Service is down"
          description: "ERP application has been down for 1 minute"

      - alert: HighPodCPU
        expr: rate(container_cpu_usage_seconds_total[5m]) > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage in pod {{ $labels.pod }}"

      - alert: PodCrashLooping
        expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Pod {{ $labels.pod }} is crash looping"

      - alert: DeploymentReplicasMismatch
        expr: kube_deployment_spec_replicas != kube_deployment_status_replicas_available
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Deployment {{ $labels.deployment }} replicas mismatch"
```

---

## FinOps — Kubernetes Cost Optimisation

### Resource Quotas per Namespace

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "50"
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: staging-quota
  namespace: staging
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "25"
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "5"
    requests.memory: 10Gi
    limits.cpu: "10"
    limits.memory: 20Gi
    pods: "15"
```

### Cost Optimisation Results

| Strategy | Monthly Saving |
|---|---|
| Right-sized pod resources | $400 |
| Dev namespace auto-shutdown | $300 |
| Spot instances for non-prod | $500 |
| Unused image cleanup | $100 |
| **Total** | **$1,300/month** |

---

## Container Security

### Security Scanning Pipeline

```yaml
# Trivy security scan in CI/CD
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ env.IMAGE }}
    format: table
    exit-code: '1'
    ignore-unfixed: true
    severity: CRITICAL,HIGH
```

### Kubernetes Network Policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: erp-app-netpol
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: erp-app
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: production
      ports:
        - protocol: TCP
          port: 3000
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: production
      ports:
        - protocol: TCP
          port: 5432
```

---

## Standards & Frameworks Referenced

- **CNCF Cloud Native Landscape** — Cloud native best practices
- **CIS Kubernetes Benchmark** — Security hardening
- **Docker Security Best Practices** — Container security
- **GitOps Principles** — Infrastructure and deployment management
- **DORA Metrics** — DevOps performance measurement
- **FinOps Foundation** — Cloud cost optimisation

---

## Tools & Technologies

![Docker](https://img.shields.io/badge/Docker-Containerization-blue)
![Kubernetes](https://img.shields.io/badge/Kubernetes-Orchestration-blue)
![Terraform](https://img.shields.io/badge/Terraform-IaC-purple)
![GitHub Actions](https://img.shields.io/badge/GitHub-Actions-black)
![Jenkins](https://img.shields.io/badge/Jenkins-Pipeline-red)
![Prometheus](https://img.shields.io/badge/Prometheus-Monitoring-orange)
![Grafana](https://img.shields.io/badge/Grafana-Dashboards-orange)
![AWS EKS](https://img.shields.io/badge/AWS-EKS-orange)
![Trivy](https://img.shields.io/badge/Trivy-Security%20Scan-red)
![Helm](https://img.shields.io/badge/Helm-Package%20Manager-blue)

---

## Author

**George Amankwaa Sarpong**
Cloud & DevOps Engineer | Kubernetes & Container Specialist
📍 Accra, Ghana
🔗 [LinkedIn](https://linkedin.com/in/georgesarpong)
🌐 [GitHub Portfolio](https://github.com/GeorgeSarpong)

---

*This project is part of a broader portfolio demonstrating readiness for Cloud DevOps Engineer and Cloud Infrastructure Engineer roles in the US and global market.*
