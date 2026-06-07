# Implementation Plan: Infrastructure & Deployment
**Antigravity IDE — Open WebUI SaaS Layer**
**Document Version:** 1.0
**Focus Area:** Kubernetes deployment, auto-scaling, cloud infrastructure, DevOps

---

## Overview

A SaaS system is only as good as its infrastructure. This plan covers deploying the full stack — Open WebUI, LiteLLM, PostgreSQL, Redis, Caddy — on Kubernetes for reliability, auto-scaling, and multi-region failover. We'll use IaC (Infrastructure as Code) to make everything reproducible and version-controlled.

---

## Target Deployment Architecture

```
Internet Traffic
        │
        ▼
   Cloudflare / AWS CloudFront (DDoS, caching)
        │
        ▼
   AWS Application Load Balancer (SSL termination)
        │
        ▼
   Kubernetes Cluster (EKS or self-hosted)
        │
   ┌────┴────────────────────────────────┐
   │    Ingress Controller (Caddy/Traefik) │
   │    (custom domain routing, SSL)       │
   └────┬────────────────────────────────┘
        │
   ┌────┴──────────────────────────┐
   │  Microservices / Workloads:   │
   │  - Open WebUI (3+ replicas)   │
   │  - LiteLLM Proxy (2+ replicas)│
   │  - Workers (Celery/ARQ)       │
   │  - Redis                      │
   │  - Postgres Primary           │
   │  - Postgres Replica(s)        │
   │  - Prometheus (metrics)       │
   │  - Loki (logs)                │
   └───────────────────────────────┘
```

---

## Phase 1: Kubernetes Cluster Setup

### 1.1 EKS (AWS) — Production Recommended

Using Terraform to provision:

```hcl
# infra/terraform/eks.tf

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.24"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# EKS Cluster
resource "aws_eks_cluster" "antigravity" {
  name            = "antigravity-saas"
  role_arn        = aws_iam_role.eks_cluster.arn
  version         = "1.28"
  
  vpc_config {
    subnet_ids = [
      aws_subnet.private_a.id,
      aws_subnet.private_b.id,
      aws_subnet.private_c.id,
    ]
    security_groups = [aws_security_group.cluster.id]
    endpoint_private_access = true
    endpoint_public_access  = true
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_policy,
  ]
}

# Node Group (auto-scaling)
resource "aws_eks_node_group" "application" {
  cluster_name    = aws_eks_cluster.antigravity.name
  node_group_name = "application-nodes"
  node_role_arn   = aws_iam_role.node_group.arn
  
  subnet_ids = [
    aws_subnet.private_a.id,
    aws_subnet.private_b.id,
    aws_subnet.private_c.id,
  ]

  scaling_config {
    desired_size = 3
    max_size     = 10
    min_size     = 3
  }

  instance_types = ["t3.xlarge"]     # 4 CPU, 16 GB RAM per node
  disk_size      = 100

  tags = {
    Environment = "production"
    Workload    = "application"
  }
}

# GPU Node Group (for Ollama if using Kubernetes)
resource "aws_eks_node_group" "gpu" {
  cluster_name    = aws_eks_cluster.antigravity.name
  node_group_name = "gpu-nodes"
  node_role_arn   = aws_iam_role.node_group.arn
  
  subnet_ids = [aws_subnet.private_a.id]

  scaling_config {
    desired_size = 1
    max_size     = 3
    min_size     = 1
  }

  instance_types = ["g4dn.2xlarge"]  # 1x NVIDIA T4 GPU, 8 CPU, 32 GB RAM
  disk_size      = 200

  tags = {
    Environment = "production"
    Workload    = "gpu"
  }
}

# Auto Scaling
resource "aws_autoscaling_group" "application" {
  name = "antigravity-asg"
  vpc_zone_identifier = [
    aws_subnet.private_a.id,
    aws_subnet.private_b.id,
  ]
  
  min_size         = 3
  max_size         = 10
  desired_capacity = 3
  health_check_type = "ELB"
  
  tag {
    key                 = "k8s.io/cluster-autoscaler/antigravity-saas"
    value               = "owned"
    propagate_at_launch = false
  }
}

output "cluster_endpoint" {
  value = aws_eks_cluster.antigravity.endpoint
}

output "cluster_name" {
  value = aws_eks_cluster.antigravity.name
}
```

### 1.2 Multi-Zone Deployment

Deploy across 3 AZs for high availability:
- Control plane: 3 replicas across AZs (AWS-managed, auto-HA)
- Application nodes: spread across AZs via Kubernetes pod affinity rules
- Database: primary in AZ-a, read replica in AZ-b, backup in AZ-c

---

## Phase 2: Kubernetes Manifests (Helm Charts)

### 2.1 Helm Chart Structure

```
charts/
├── open-webui/
│   ├── Chart.yaml
│   ├── values.yaml             # default values
│   ├── values-prod.yaml        # production overrides
│   ├── templates/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── configmap.yaml
│   │   ├── secret.yaml
│   │   ├── pvc.yaml
│   │   ├── hpa.yaml            # horizontal pod autoscaler
│   │   └── podmonitor.yaml     # prometheus metrics
│   └── README.md
├── litellm-proxy/
├── postgresql/
├── redis/
└── monitoring/
```

### 2.2 Open WebUI Deployment

`charts/open-webui/templates/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "open-webui.fullname" . }}
  labels:
    {{- include "open-webui.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "open-webui.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "open-webui.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "open-webui.serviceAccountName" . }}
      
      # Anti-affinity: spread pods across different nodes
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values: [open-webui]
                topologyKey: kubernetes.io/hostname

      containers:
      - name: open-webui
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: IfNotPresent
        
        ports:
        - containerPort: 8080
          name: http

        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: {{ include "open-webui.fullname" . }}-db
              key: url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: {{ include "open-webui.fullname" . }}-redis
              key: url
        - name: OPENAI_API_BASE_URL
          value: "http://litellm-proxy:4000"
        - name: STRIPE_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: {{ include "open-webui.fullname" . }}-stripe
              key: secret-key
        {{- range $key, $value := .Values.env }}
        - name: {{ $key }}
          value: {{ $value | quote }}
        {{- end }}

        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 2000m
            memory: 2Gi

        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3

        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3

        volumeMounts:
        - name: config
          mountPath: /app/config
          readOnly: true

      volumes:
      - name: config
        configMap:
          name: {{ include "open-webui.fullname" . }}-config
```

### 2.3 Horizontal Pod Autoscaler

`charts/open-webui/templates/hpa.yaml`:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "open-webui.fullname" . }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "open-webui.fullname" . }}
  minReplicas: 3
  maxReplicas: 20
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
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
      - type: Pods
        value: 2
        periodSeconds: 30
      selectPolicy: Max
```

---

## Phase 3: PostgreSQL on Kubernetes

### 3.1 StatefulSet Deployment

For production, **don't use** a Kubernetes-native PostgreSQL operator (they're complex). Instead:
- Use **AWS RDS** (managed) — simpler, automatic backups, failover
- Or self-host with **Helm chart** (Bitnami) for more control

**Recommended: AWS RDS**

```hcl
# infra/terraform/rds.tf

resource "aws_db_instance" "postgres_primary" {
  identifier              = "antigravity-postgres"
  engine                  = "postgres"
  engine_version          = "16.0"
  instance_class          = "db.r6i.xlarge"
  allocated_storage       = 100
  max_allocated_storage   = 500       # auto-scale up to 500GB
  
  db_name                 = "openwebui"
  username                = "owui"
  password                = random_password.db_password.result
  
  multi_az                = true
  publicly_accessible     = false
  vpc_security_group_ids  = [aws_security_group.postgres.id]
  db_subnet_group_name    = aws_db_subnet_group.private.name
  
  backup_retention_period = 30
  backup_window          = "03:00-04:00"
  maintenance_window     = "mon:04:00-mon:05:00"
  
  skip_final_snapshot    = false
  final_snapshot_identifier = "antigravity-postgres-final-snapshot-${formatdate("YYYY-MM-DD-hhmm", timestamp())}"
  
  enabled_cloudwatch_logs_exports = ["postgresql"]
  
  tags = {
    Environment = "production"
    Service     = "openwebui"
  }
}

resource "aws_db_instance" "postgres_replica" {
  identifier            = "antigravity-postgres-replica"
  replicate_source_db   = aws_db_instance.postgres_primary.identifier
  instance_class        = "db.r6i.large"
  
  publicly_accessible   = false
  skip_final_snapshot   = true
  
  availability_zone     = data.aws_availability_zones.available.names[1]
  
  tags = {
    Environment = "production"
    Service     = "openwebui"
  }
}

output "postgres_endpoint" {
  value = aws_db_instance.postgres_primary.endpoint
}

output "postgres_replica_endpoint" {
  value = aws_db_instance.postgres_replica.endpoint
}
```

---

## Phase 4: Redis Deployment

### 4.1 AWS ElastiCache

```hcl
# infra/terraform/elasticache.tf

resource "aws_elasticache_cluster" "redis" {
  cluster_id           = "antigravity-redis"
  engine               = "redis"
  engine_version       = "7.0"
  node_type            = "cache.r6g.xlarge"
  num_cache_nodes      = 1
  parameter_group_name = "default.redis7"
  port                 = 6379
  
  security_group_ids   = [aws_security_group.redis.id]
  subnet_group_name    = aws_elasticache_subnet_group.private.name
  
  automatic_failover_enabled = true
  multi_az_enabled           = true
  
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = random_password.redis_auth.result
  
  snapshot_retention_limit = 5
  snapshot_window         = "03:00-05:00"
  
  cloudwatch_log_group_names = [aws_cloudwatch_log_group.redis.name]
  
  tags = {
    Environment = "production"
  }
}

resource "aws_elasticache_replication_group" "redis" {
  replication_group_description = "Antigravity Redis Cluster"
  engine                        = "redis"
  engine_version                = "7.0"
  node_type                     = "cache.r6g.xlarge"
  num_cache_clusters            = 3
  automatic_failover_enabled    = true
  multi_az_enabled              = true
  
  port                   = 6379
  security_group_ids     = [aws_security_group.redis.id]
  subnet_group_name      = aws_elasticache_subnet_group.private.name
  
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = random_password.redis_auth.result
  
  cloudwatch_log_group_names = [aws_cloudwatch_log_group.redis.name]
  
  tags = {
    Environment = "production"
  }
}
```

---

## Phase 5: Ingress & Load Balancing

### 5.1 Kubernetes Ingress with Caddy/Traefik

Using Traefik for advanced routing (custom domains, tenant resolution):

```yaml
# charts/traefik-ingress/values-prod.yaml

traefik:
  image:
    name: traefik
    tag: "2.10"
  
  deployment:
    replicas: 3
    tolerations: []
    affinity:
      podAntiAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values: [traefik]
            topologyKey: kubernetes.io/hostname
  
  service:
    type: LoadBalancer
    spec:
      type: LoadBalancer
      loadBalancerSourceRanges:
      - 0.0.0.0/0
  
  # DNS challenge for Let's Encrypt
  acme:
    enabled: true
    email: ops@antigravity.dev
    storage: /data/acme.json
    caServer: https://acme-v02.api.letsencrypt.org/directory
    tlsChallenge: {}
    dnsChallenge:
      provider: route53
      resolvers:
      - 1.1.1.1
      - 8.8.8.8

  # Custom domains routing
  ingressRoute:
    main:
      entryPoints:
      - websecure
      routes:
      - match: "Host(`yoursaas.com`, `*.yoursaas.com`, `*.subdomain.yoursaas.com`)"
        kind: Rule
        services:
        - name: open-webui
          port: 8080
        middlewares:
        - name: tenant-header-injection
```

### 5.2 AWS ALB for Ingress

If using AWS ALB directly (instead of Traefik):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: open-webui-main
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/certificate-arn: "arn:aws:acm:region:account:certificate/id"
spec:
  rules:
  - host: yoursaas.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: open-webui
            port:
              number: 8080
  - host: "*.yoursaas.com"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: open-webui
            port:
              number: 8080
```

---

## Phase 6: Deployment Pipeline (CI/CD)

### 6.1 GitHub Actions Workflow

`.github/workflows/deploy.yml`:

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write

    steps:
    - uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions-role
        aws-region: ap-southeast-1

    - name: Login to ECR
      run: |
        aws ecr get-login-password --region ap-southeast-1 | \
        docker login --username AWS --password-stdin ${{ secrets.ECR_REGISTRY }}

    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: |
          ${{ secrets.ECR_REGISTRY }}/open-webui:latest
          ${{ secrets.ECR_REGISTRY }}/open-webui:${{ github.sha }}
        cache-from: type=registry,ref=${{ secrets.ECR_REGISTRY }}/open-webui:buildcache
        cache-to: type=registry,ref=${{ secrets.ECR_REGISTRY }}/open-webui:buildcache,mode=max

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Run tests
      run: |
        docker run \
          --env DATABASE_URL="postgres://test:test@localhost/test" \
          -v /var/run/docker.sock:/var/run/docker.sock \
          ${{ secrets.ECR_REGISTRY }}/open-webui:${{ github.sha }} \
          python -m pytest tests/ -v

  deploy:
    needs: [build, test]
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
    - uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions-role
        aws-region: ap-southeast-1

    - name: Update kubeconfig
      run: |
        aws eks update-kubeconfig \
          --name antigravity-saas \
          --region ap-southeast-1

    - name: Helm deploy
      run: |
        helm upgrade --install open-webui ./charts/open-webui \
          --namespace production \
          --create-namespace \
          -f ./charts/open-webui/values-prod.yaml \
          --set image.tag=${{ github.sha }} \
          --wait \
          --timeout 5m

    - name: Verify deployment
      run: |
        kubectl rollout status deployment/open-webui -n production --timeout=5m
        kubectl get svc,pod -n production

    - name: Slack notification
      if: always()
      uses: slackapi/slack-github-action@v1
      with:
        payload: |
          {
            "text": "Deployment ${{ job.status }}",
            "channel": "#ops-alerts"
          }
```

---

## Phase 7: Secrets Management

### 7.1 AWS Secrets Manager

```hcl
# infra/terraform/secrets.tf

resource "aws_secretsmanager_secret" "database_url" {
  name                    = "antigravity/database-url"
  recovery_window_in_days = 7
}

resource "aws_secretsmanager_secret_version" "database_url" {
  secret_id = aws_secretsmanager_secret.database_url.id
  secret_string = "postgresql://owui:${random_password.db_password.result}@${aws_db_instance.postgres_primary.endpoint}/openwebui"
}

resource "aws_secretsmanager_secret" "stripe_api_key" {
  name = "antigravity/stripe-secret-key"
}

resource "aws_secretsmanager_secret_version" "stripe_api_key" {
  secret_id     = aws_secretsmanager_secret.stripe_api_key.id
  secret_string = var.stripe_secret_key
}
```

### 7.2 Kubernetes External Secrets Operator

```yaml
# Sync secrets from AWS Secrets Manager to Kubernetes
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-southeast-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: open-webui-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets
    kind: SecretStore
  target:
    name: open-webui-secrets
    creationPolicy: Owner
  data:
  - secretKey: database-url
    remoteRef:
      key: antigravity/database-url
  - secretKey: stripe-secret-key
    remoteRef:
      key: antigravity/stripe-secret-key
  - secretKey: redis-url
    remoteRef:
      key: antigravity/redis-url
```

---

## Phase 8: Backup & Disaster Recovery

### 8.1 Database Backups

AWS RDS auto-handles backups (30-day retention).

For critical data, also:
```bash
# Weekly PostgreSQL dump to S3
#!/bin/bash
BACKUP_DATE=$(date +%Y-%m-%d-%H%M%S)
pg_dump \
  --host $DB_HOST \
  --username $DB_USER \
  --no-password \
  --format=custom \
  openwebui | \
  aws s3 cp - s3://antigravity-backups/postgres/backup-$BACKUP_DATE.dump

# Retention: keep for 90 days
aws s3 rm s3://antigravity-backups/postgres \
  --recursive \
  --exclude "*" \
  --include "backup-*.dump" \
  --delete-older-than 90d
```

### 8.2 Disaster Recovery Plan

| RTO | RPO | Scenario | Recovery | Cost |
|---|---|---|---|---|
| 1 hour | 15 min | DB failure | Promote read replica (AWS RDS) | Low |
| 4 hours | 1 hour | Region down | Restore from backup in other region | Medium |
| 24 hours | 8 hours | Complete data loss | Restore from S3 backup dump | Low |

---

## Environment File Structure

```bash
# .env.production

# Kubernetes
KUBE_CLUSTER=antigravity-saas
KUBE_REGION=ap-southeast-1
KUBE_NAMESPACE=production

# Container Registry
ECR_REGISTRY=123456789.dkr.ecr.ap-southeast-1.amazonaws.com

# Database
DATABASE_URL=postgresql://owui:${DB_PASSWORD}@antigravity-postgres.rds.amazonaws.com:5432/openwebui
REPLICA_DATABASE_URLS=postgresql://owui:${DB_PASSWORD}@antigravity-postgres-replica.rds.amazonaws.com:5432/openwebui

# Redis
REDIS_URL=redis://:${REDIS_AUTH}@antigravity-redis.ng.0001.apse1.cache.amazonaws.com:6379

# Stripe
STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...

# AWS S3
S3_BUCKET_NAME=antigravity-assets
S3_REGION=ap-southeast-1
CDN_BASE_URL=https://assets.yoursaas.com

# Monitoring
PROMETHEUS_URL=http://prometheus.monitoring:9090
LOKI_URL=http://loki.monitoring:3100

# Slack for alerting
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/...
```

---

## Implementation Checklist

- [ ] EKS cluster created (or alternative: GKE, DigitalOcean)
- [ ] VPC, subnets, security groups configured
- [ ] RDS PostgreSQL primary + replica deployed
- [ ] ElastiCache Redis cluster deployed
- [ ] Helm charts created for all services
- [ ] HPA configured (min 3, max 20 replicas for Open WebUI)
- [ ] ArgoCD or Flux deployed for GitOps CD
- [ ] GitHub Actions pipeline deployed
- [ ] AWS Secrets Manager populated with all secrets
- [ ] External Secrets Operator syncing secrets to K8s
- [ ] Ingress controller (Traefik/ALB) routing traffic correctly
- [ ] SSL/TLS certificates auto-renewing (Let's Encrypt)
- [ ] CloudFront distribution in front of ALB for DDoS + caching
- [ ] Database backups to S3 scheduled daily
- [ ] Monitoring (Prometheus/Grafana) deployed
- [ ] Logging (Loki/ELK) deployed
- [ ] Deployment successful with rolling updates (0 downtime)
