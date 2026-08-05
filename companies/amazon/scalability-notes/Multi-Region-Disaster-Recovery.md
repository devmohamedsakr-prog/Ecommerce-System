# Amazon Scalability: Multi-Region Disaster Recovery

## Infrastructure Philosophy
- Availability > Cost: Spend extra for reliability
- Multi-region design: No single point of failure
- Active-active: Not standby (all regions live)
- Rapid failover: Automatic (seconds, not hours)

## Primary Region Failure Scenario

### Before Failure
- US-East (Virginia): Primary (60% traffic)
- US-West (Oregon): Secondary (20% traffic)
- EU-West (Ireland): Tertiary (10% traffic)
- Asia-Pacific: Distributed (10% traffic)

### Failure Detection
- Health check: Every 5 seconds
- Failure detection: After 2 consecutive failures (10 seconds)
- Automated response: Immediately triggered

### Traffic Rerouting
- DNS failover: Updated in seconds
- Load balancer: Routes traffic to remaining regions
- Database: Read replicas become primary (if writable)
- Result: 99%+ orders not affected (already in-flight)

### Recovery Time Objective (RTO)
- Detection: 10 seconds
- Failover: 10 seconds
- Full traffic rerouted: 30 seconds total
- SLA: 99.99% uptime (52 minutes downtime/year max)

## Database Strategy
- DynamoDB: Multi-master replication
  - Write capacity: Spread across regions
  - Reads: Local (latency <100ms per region)
  - Conflict resolution: Last-write-wins

- RDS (SQL):
  - Primary: US-East (single master)
  - Read replicas: 3+ per region (hot standbys)
  - Promotion: Automatic if primary fails
  - Data loss: <1 minute (RPO)

## Failover Testing
- Monthly: Simulate region failure
- Test: Full traffic reroute scenario
- Load testing: Verify remaining regions handle surge
- Result: Confidence in failover process

## Cost Analysis
- Single region: Baseline (lowest cost)
- Multi-region: +30% infrastructure cost
- Redundancy: Failover databases cost
- Justification: Downtime > $10M/minute revenue

## Global Scale Example
- If US-East fails:
  - Customers: Rerouted to other regions
  - Orders: Continue flowing (backup centers)
  - Delivery: On-time maintained (stock in multiple FCs)
  - Revenue: <0.1% impact
  - Result: Invisible to customers

## Future Resilience
- Edge computing: Reduce latency
- Multi-cloud: Diversify beyond AWS
- Autonomous failover: ML predicts failures before occur
- Result: Approach 99.999% availability (5 min/year downtime)

