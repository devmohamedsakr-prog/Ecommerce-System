# Deployment & Infrastructure Guide

## 📋 Overview

Complete guide to deploying e-commerce systems from MVP to enterprise scale.

## 🚀 Phase 1: MVP Deployment (Heroku/Simple VPS)

### Heroku Deployment

```bash
# 1. Create Heroku app
heroku create ecommerce-api

# 2. Set environment variables
heroku config:set NODE_ENV=production
heroku config:set DATABASE_URL=postgresql://...
heroku config:set JWT_SECRET=...

# 3. Add PostgreSQL addon
heroku addons:create heroku-postgresql:standard-0

# 4. Deploy
git push heroku main

# 5. Run migrations
heroku run npm run migrate

# Cost: $50-100/month
# Scalability: Up to 10k users
# Effort: Minimal (managed service)
```

### Docker + DigitalOcean

```dockerfile
# Dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

```bash
# Build image
docker build -t ecommerce-api:1.0 .

# Deploy to DigitalOcean
doctl compute droplet create ecommerce-api \
  --image docker-20-10-12 \
  --size s-2vcpu-4gb \
  --region nyc3

# Cost: $24/month (single droplet)
# Scalability: Up to 50k users
# Effort: Moderate (you manage everything)
```

## 🎯 Phase 2: Scaling Deployment (AWS)

### AWS Architecture

```
┌──────────────────────────────────────┐
│ Route 53 (DNS)                       │
└──────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────┐
│ CloudFront (CDN)                     │
└──────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────┐
│ Application Load Balancer (ALB)      │
└──────────────────────────────────────┘
             │
    ┌────────┴────────┬────────────┐
    ▼                 ▼            ▼
┌────────┐      ┌────────┐    ┌────────┐
│EC2     │      │EC2     │    │EC2     │
│Instance│      │Instance│    │Instance│
│Auto    │      │Auto    │    │Auto    │
│Scaling │      │Scaling │    │Scaling │
└────────┘      └────────┘    └────────┘
    │                 │            │
    └────────┬────────┴────────────┘
             ▼
┌──────────────────────────────────────┐
│ RDS (PostgreSQL)                     │
│ Master + Read Replicas               │
└──────────────────────────────────────┘
             │
    ┌────────┴──────────┬──────────┐
    ▼                   ▼          ▼
ElastiCache    S3        Backup   Replica
(Redis)
```

### Terraform Infrastructure as Code

```hcl
# main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
}

# Public Subnets
resource "aws_subnet" "public_1a" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"
}

resource "aws_subnet" "public_1b" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "us-east-1b"
}

# RDS Instance
resource "aws_db_instance" "main" {
  identifier     = "ecommerce-db"
  engine         = "postgres"
  engine_version = "14.0"
  instance_class = "db.t3.medium"
  allocated_storage = 20
  
  db_name  = "ecommerce"
  username = "admin"
  password = var.db_password
  
  multi_az               = true
  publicly_accessible   = false
  skip_final_snapshot   = false
  
  tags = {
    Name = "ecommerce-db"
  }
}

# ALB
resource "aws_lb" "main" {
  name               = "ecommerce-alb"
  internal           = false
  load_balancer_type = "application"
  
  subnets = [
    aws_subnet.public_1a.id,
    aws_subnet.public_1b.id
  ]
}

# Auto Scaling Group
resource "aws_autoscaling_group" "main" {
  name                = "ecommerce-asg"
  vpc_zone_identifier = [aws_subnet.public_1a.id, aws_subnet.public_1b.id]
  
  min_size         = 3
  max_size         = 10
  desired_capacity = 5
  
  launch_template {
    id      = aws_launch_template.main.id
    version = "$Latest"
  }
}

# Deploy
# terraform init
# terraform plan
# terraform apply
```

## 🐳 Phase 3: Microservices (Kubernetes)

### Kubernetes Deployment

```yaml
# order-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  labels:
    app: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: ecommerce/order-service:1.2.0
        ports:
        - containerPort: 3001
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        - name: REDIS_URL
          valueFrom:
            configMapKeyRef:
              name: service-config
              key: redis_url
        livenessProbe:
          httpGet:
            path: /health
            port: 3001
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3001
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi

---
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3001
  type: ClusterIP

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Helm Chart (Package Manager)

```yaml
# Chart.yaml
apiVersion: v2
name: ecommerce-api
version: 1.0.0
appVersion: "1.0"

---
# values.yaml
replicaCount: 3

image:
  repository: ecommerce/api
  tag: "1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 3000

ingress:
  enabled: true
  className: "nginx"
  hosts:
  - host: api.example.com
    paths:
    - path: /
      pathType: Prefix

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

---
# Deployment via Helm
helm install ecommerce-api ./chart -n production
helm upgrade ecommerce-api ./chart -n production
```

## 🔄 CI/CD Pipeline

### GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node
      uses: actions/setup-node@v3
      with:
        node-version: '18'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run tests
      run: npm run test:all
    
    - name: Build Docker image
      if: github.ref == 'refs/heads/main'
      run: |
        docker build -t ecommerce/api:${{ github.sha }} .
        docker tag ecommerce/api:${{ github.sha }} ecommerce/api:latest
    
    - name: Push to Docker Hub
      if: github.ref == 'refs/heads/main'
      run: |
        echo ${{ secrets.DOCKER_PASSWORD }} | docker login -u ${{ secrets.DOCKER_USERNAME }} --password-stdin
        docker push ecommerce/api:${{ github.sha }}
        docker push ecommerce/api:latest
    
    - name: Deploy to Kubernetes
      if: github.ref == 'refs/heads/main'
      run: |
        kubectl set image deployment/order-service \
          order-service=ecommerce/api:${{ github.sha }} \
          -n production
        kubectl rollout status deployment/order-service -n production
```

## 📊 Database Migrations

### Liquibase for Schema Management

```xml
<!-- migrations/001-initial-schema.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
  xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
  http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.1.xsd">

  <changeSet id="1" author="team">
    <createTable tableName="orders">
      <column name="id" type="UUID" defaultValue="gen_random_uuid()">
        <constraints primaryKey="true"/>
      </column>
      <column name="customer_id" type="UUID">
        <constraints nullable="false"/>
      </column>
      <column name="total" type="DECIMAL(12,2)">
        <constraints nullable="false"/>
      </column>
      <column name="status" type="VARCHAR(50)">
        <constraints nullable="false"/>
      </column>
      <column name="created_at" type="TIMESTAMP" defaultValueDate="now()">
        <constraints nullable="false"/>
      </column>
    </createTable>
    
    <createIndex tableName="orders" indexName="idx_orders_customer">
      <column name="customer_id"/>
    </createIndex>
  </changeSet>

  <changeSet id="2" author="team">
    <addColumn tableName="orders">
      <column name="notes" type="TEXT"/>
    </addColumn>
  </changeSet>
</databaseChangeLog>
```

## 🔒 Secrets Management

### AWS Secrets Manager

```bash
# Create secret
aws secretsmanager create-secret \
  --name ecommerce/db-password \
  --secret-string "MySecurePassword123!"

# Use in Kubernetes
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  password: base64-encoded-password
```

## 📈 Monitoring & Alerts

### CloudWatch Alarms

```bash
# High CPU alert
aws cloudwatch put-metric-alarm \
  --alarm-name "ecommerce-high-cpu" \
  --alarm-description "Alert when CPU > 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:...

# Database connection alert
aws cloudwatch put-metric-alarm \
  --alarm-name "ecommerce-db-connections" \
  --alarm-description "Alert when connections > 100" \
  --metric-name DatabaseConnections \
  --namespace AWS/RDS \
  --statistic Average \
  --period 300 \
  --threshold 100 \
  --comparison-operator GreaterThanThreshold
```

## ✅ Deployment Checklist

```
Pre-Deployment:
- [ ] All tests passing
- [ ] Code reviewed and approved
- [ ] Database migrations tested
- [ ] Environment variables configured
- [ ] Secrets secured
- [ ] Monitoring alerts active
- [ ] Rollback plan documented

Deployment:
- [ ] Backup database
- [ ] Run migrations
- [ ] Deploy new code
- [ ] Health checks passing
- [ ] Monitor error rates
- [ ] Performance metrics normal
- [ ] Customer reports: no issues

Post-Deployment:
- [ ] Smoke tests passing
- [ ] Analytics showing expected traffic
- [ ] No error spikes
- [ ] Performance within SLA
- [ ] Celebrate 🎉
- [ ] Document any issues
- [ ] Plan follow-up improvements
```

---

**Key Principle:** "Automate everything. Manual deployments are the enemy of reliability."
