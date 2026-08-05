# DevOps & Deployment Best Practices

**Status:** Best Practice Guide | **Priority:** CRITICAL | **Target:** 50+ deployments/day safely

---

## Overview

DevOps and CI/CD enable rapid, safe deployments. Manual deployments error-prone and slow. Automated pipelines: 50+ deployments/day with 0.1% rollback rate. Infrastructure as Code: reproducible, versioned, auditable infrastructure.

## Business Impact

- Manual deployment: 1/week, high risk, 2-hour process
- Automated deployment: 50/day, low risk, 5-minute process
- Result: 10x faster innovation, 100x fewer manual errors
- Cost: Infrastructure-as-code catches 80% of misconfigurations

## CI/CD Pipeline Stages

### Stage 1: Code Commit
```
Developer pushes code to GitHub
Webhook triggers CI/CD pipeline
Build starts automatically

Benefits:
- No manual intervention
- Immediate feedback
- Fast iteration
```

### Stage 2: Build & Test
```
Run immediately:
1. Unit tests (1-5 minutes)
2. Lint/static analysis (< 1 minute)
3. Build Docker image (2-5 minutes)
4. Security scanning (< 2 minutes)

Total time: 5-15 minutes
Failure: Halt deployment, alert developer
Success: Proceed to staging
```

### Stage 3: Integration Tests
```
Deploy to staging environment
Run integration tests (10-30 minutes)
Test against real services
Catch environment-specific bugs

Failure: Halt, alert developer
Success: Proceed to canary
```

### Stage 4: Canary Deployment
```
Deploy to production (1-5% of servers)
Monitor for errors (5-10 minutes)
If error rate spike: Rollback
If stable: Gradually increase to 50%
Then: Complete rollout

Risk: Only 1-5% of users affected if issue
Benefit: Real production testing before full rollout
```

### Stage 5: Full Deployment
```
All servers updated
Monitor metrics:
- Error rate
- Latency
- Traffic distribution

Automated rollback if issues detected
```

## Infrastructure as Code (IaC)

### Terraform Example

```hcl
# Define ecommerce infrastructure

# Database
resource "aws_rds_instance" "postgres" {
  identifier     = "ecommerce-db"
  engine         = "postgres"
  instance_class = "db.t3.medium"
  allocated_storage = 100
  
  backup_retention_period = 30
  multi_az                = true
}

# Load Balancer
resource "aws_elb" "main" {
  name = "ecommerce-lb"
  
  listener {
    instance_port     = 3000
    instance_protocol = "http"
    lb_port           = 80
    lb_protocol       = "http"
  }
  
  health_check {
    healthy_threshold   = 2
    unhealthy_threshold = 2
    timeout             = 3
    interval            = 30
    target              = "HTTP:3000/health"
  }
}

# Autoscaling
resource "aws_autoscaling_group" "main" {
  launch_configuration = aws_launch_configuration.main.id
  min_size             = 3
  max_size             = 20
  load_balancers       = [aws_elb.main.name]
  
  tag {
    key                 = "Name"
    value               = "ecommerce-server"
    propagate_launch_configuration = true
  }
}

# Define scaling policies
resource "aws_autoscaling_policy" "scale_up" {
  name                   = "scale-up"
  scaling_adjustment     = 2
  adjustment_type        = "ChangeInCapacity"
  autoscaling_group_name = aws_autoscaling_group.main.name
  cooldown               = 300
}

resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  alarm_name          = "cpu-high"
  alarm_actions       = [aws_autoscaling_policy.scale_up.arn]
  metric_name         = "CPUUtilization"
  threshold           = 70
  evaluation_periods  = 2
  period              = 60
}
```

Benefits:
- Infrastructure versioned in Git
- Changes reviewed before deploy
- Reproducible across environments
- Disaster recovery: Rebuild infrastructure in minutes
- Cost tracking: Every resource has version history

## Kubernetes Deployment Strategy

### Blue-Green Deployment

```
1. Current production: Blue (100% traffic)
2. New version: Green (0% traffic, running)
3. All tests pass on Green
4. Switch: Route 100% traffic to Green
5. Rollback: Simple switch back to Blue

Risk: Minimal (switch is instant)
Downtime: Zero (both running)
Test: Full production test before cutover
```

### Rolling Deployment

```
1. Service has 5 pods running v1
2. Replace 1 pod with v2
3. If healthy: Continue replacing
4. If 5 pods v2 healthy: Complete
5. If issue detected at step 2: Rollback

Advantage: No extra resources needed
Trade-off: Temporary mixed versions
```

### Canary Deployment (Kubernetes)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
  - order-service
  http:
  - match:
    - uri:
      prefix: "/api"
    route:
    - destination:
        host: order-service
        subset: v1
      weight: 95  # 95% to stable
    - destination:
        host: order-service
        subset: v2
      weight: 5   # 5% to canary
    timeout: 10s
    retries:
      attempts: 3
```

## Monitoring Deployments

### Pre-Deployment Checklist
```
✓ All tests passing
✓ Code reviewed and approved
✓ Migrations tested in staging
✓ Monitoring alerts configured
✓ Rollback plan documented
✓ On-call engineer available
```

### Deployment Metrics
```
Track during deployment:
- Error rate (alert if > 1%)
- Latency p99 (alert if > 2x baseline)
- Throughput (alert if < 90% baseline)
- Pod restart rate (alert if > 0%)

Automated rollback if:
- Error rate > 5%
- Latency spike > 300%
- Service unavailable
```

### Post-Deployment
```
Monitor for 1 hour:
- Error rates stable
- Performance normal
- Resource usage stable
- No customer complaints

Sign-off: Deployment successful
```

## Disaster Recovery

### Backup Strategy
```
Automated daily snapshots:
- Database backups (compressed)
- Configuration backups
- Code repository backups

Recovery time objective (RTO):
- Minor issue: 5 minutes (rollback)
- Major issue: 1 hour (restore from backup)
- Catastrophic: 4 hours (rebuild infrastructure)

Test: Quarterly recovery drills
Verify backups restore successfully
```

### Incident Response
```
1. Detect issue (automated alert)
2. Acknowledge incident (within 1 min)
3. Assess impact (within 5 min)
4. Decide: Rollback or fix forward
5. Execute decision (within 5 min)
6. Verify stability (5-15 min)
7. Post-mortem (within 24 hours)

Target RTO: 15 minutes
Target RPO: 5 minutes (max data loss)
```

## Secrets Management

```
Never commit secrets to Git:
✗ database_password = "super_secret_123"
✗ api_key = "sk_live_abc123"

Use secrets manager:
✓ AWS Secrets Manager
✓ HashiCorp Vault
✓ Kubernetes Secrets

Rotation:
- Rotate secrets every 90 days
- Automated rotation for API keys
- No downtime during rotation
```

## Logging & Observability

### Centralized Logging (ELK Stack)
```
Elasticsearch: Log storage & indexing
Logstash: Log collection & transformation
Kibana: Log visualization & search

Every log entry includes:
- Timestamp
- Service name
- Log level (DEBUG, INFO, WARN, ERROR)
- Request ID (trace across services)
- User ID (PII masked)
- Message
- Stack trace (if error)

Search: Find all errors for user_id=123 in last hour
Alert: If error rate spikes
```

### Distributed Tracing (Jaeger)
```
Track request across services:
1. Client hits API Gateway (1ms)
2. API Gateway routes to Order Service (2ms)
3. Order Service calls Payment Service (45ms)
4. Payment Service calls Bank API (30ms)
5. Bank responds (30ms)
6. Response bubbles back (2ms)

Total latency: 110ms
Bottleneck identified: Bank API (30ms)
Action: Cache or optimize bank calls
```

### Monitoring (Prometheus + Grafana)
```
Metrics for every service:
- Request rate (requests/sec)
- Error rate (errors/sec)
- Latency (p50, p95, p99)
- CPU usage (%)
- Memory usage (%)
- Disk usage (%)

Alerts:
- CPU > 80% for 5 min
- Error rate > 1%
- Disk > 90%

Dashboards:
- Overview: System health
- Service-specific: Detailed metrics
- Business metrics: Revenue, conversions, checkout funnel
```

## Cost Optimization

### Resource Tagging
```
Every resource tagged:
- Environment: prod, staging, dev
- Service: order-service, payment-service
- Cost-center: ecommerce, support

Cost reports:
- Cost per service
- Cost per environment
- Cost per team
- Identify unused resources
```

### Scheduled Scaling
```
Scaling policies:
- Business hours (9-5): Min 10 servers, max 100
- Off-hours (5-9): Min 3 servers, max 20
- Weekends: Min 5 servers, max 15

Savings: 40-60% infrastructure cost
```

---

**Guide Version:** 1.0 | **Status:** Production Best Practice | **Target:** 50+ safe deployments/day

