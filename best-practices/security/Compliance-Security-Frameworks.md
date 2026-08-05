# Compliance & Security Frameworks for Enterprise E-Commerce

**Status:** Compliance Guide | **Priority:** CRITICAL | **Scope:** Enterprise certifications

---

## Framework 1: SOC 2 Type II Compliance

**What is SOC 2?**
SOC 2 certification proves your systems maintain security, availability, processing integrity, confidentiality, and privacy controls. Enterprise customers require SOC 2 before engaging.

**Business Impact:**
- Enables enterprise contracts ($1M+)
- Speeds sales cycle (compliance pre-requisite)
- Reduces security concerns
- Competitive advantage

**Five Trust Service Criteria:**

### 1. Security
Controls preventing unauthorized access to systems.

**Requirements:**
```
- Access Control
  ✓ Role-based access (RBAC)
  ✓ MFA for all staff
  ✓ Least privilege principle
  ✓ Quarterly access reviews
  ✓ Immediate removal on termination

- Change Management
  ✓ All changes to code/infrastructure tracked
  ✓ Peer review required
  ✓ Testing in staging before production
  ✓ Rollback procedures documented
  ✓ Audit trail of all changes

- Incident Response
  ✓ Documented incident response plan
  ✓ Incident severity levels defined
  ✓ 4-hour mean response time
  ✓ Public disclosure procedures
  ✓ Post-incident review (root cause analysis)

- Encryption
  ✓ HTTPS for all data in transit
  ✓ AES-256 for data at rest
  ✓ Key rotation every 90 days
  ✓ Secure key storage (vault, not code)

- Network Security
  ✓ Firewalls between environments
  ✓ DDoS protection
  ✓ WAF (Web Application Firewall)
  ✓ VPCs for isolated environments
  ✓ Regular penetration testing

Implementation:
- Tools: HashiCorp Vault (secrets), AWS VPC (networking), Datadog (monitoring)
- Effort: 3-6 months
- Cost: $50K-150K (tools + audit)
```

### 2. Availability
Systems available for operation and use as expected.

**Requirements:**
```
- Monitoring & Alerting
  ✓ 24/7 infrastructure monitoring
  ✓ Alerts for errors, latency, downtime
  ✓ On-call rotation (24/7 coverage)
  ✓ SLA: 99.9% uptime (< 45 minutes/month)

- Backup & Recovery
  ✓ Daily automated backups
  ✓ Backup testing (monthly restore drills)
  ✓ RTO: 4 hours (max time to recover)
  ✓ RPO: 1 hour (max data loss)

- Disaster Recovery Plan
  ✓ Documented procedures
  ✓ Quarterly drills
  ✓ Alternative data center failover
  ✓ Team trained on procedures

- Auto-Scaling
  ✓ Automatically scale up on demand
  ✓ Handle 10x traffic spikes
  ✓ Cost-optimized
  ✓ Performance SLA maintained

Implementation:
- Monitoring: Prometheus + Grafana
- Backup: AWS RDS automated backups + snapshots
- Failover: Multi-region or multi-AZ setup
- Cost: $30K-80K/month for redundancy
```

### 3. Processing Integrity
Processing is complete, accurate, timely, and authorized.

**Requirements:**
```
- Data Validation
  ✓ Input validation (no SQL injection)
  ✓ Data type checking
  ✓ Range validation
  ✓ Duplicate detection
  ✓ Error handling logged

- Completeness
  ✓ All transactions recorded
  ✓ No transactions lost
  ✓ Transactions processed once (idempotency)
  ✓ Sequence maintained

- Accuracy
  ✓ Calculations correct
  ✓ Rounding rules consistent
  ✓ Financial data reconciled
  ✓ Variance reports for investigation

- Authorization
  ✓ Only authorized users can modify data
  ✓ Multi-person approvals for critical changes
  ✓ Audit trail of all modifications
  ✓ Change justification required

Implementation:
- Validation: Built into APIs
- Testing: 90%+ code coverage
- Reconciliation: Daily GL reconciliation
- Audit: Complete change history in database
```

### 4. Confidentiality
Information designated as confidential is protected.

**Requirements:**
```
- Data Classification
  ✓ All data classified (public, internal, confidential, restricted)
  ✓ PII marked as restricted
  ✓ Credit card data marked as restricted
  ✓ Classification rules defined

- Access Controls
  ✓ Only authorized staff access confidential data
  ✓ Customer service reps: Can access customer data only
  ✓ Finance team: Can access GL but not customer PII
  ✓ Engineers: Access masked production data

- Encryption
  ✓ Confidential data always encrypted
  ✓ Encryption keys secured
  ✓ Key access logged

- Vendor Management
  ✓ Vendors sign NDAs
  ✓ Vendors meet security standards
  ✓ Vendor access monitored
  ✓ Vendor contracts include confidentiality

Implementation:
- Database: Column-level encryption for PII
- Access: IAM roles restrict who sees what
- Monitoring: Alert on unusual data access patterns
- Training: All staff trained on confidentiality
```

### 5. Privacy
Personal information collected, used, retained, disclosed, and destroyed per privacy laws.

**Requirements:**
```
- Privacy Policy
  ✓ Clear, detailed privacy policy
  ✓ What data collected
  ✓ Why it's collected
  ✓ Who it's shared with
  ✓ How long it's retained
  ✓ User rights (access, deletion, portability)

- Consent Management
  ✓ Explicit consent before data collection
  ✓ Consent records maintained
  ✓ Easy opt-out mechanism
  ✓ Withdrawal capability

- Data Retention
  ✓ Data deleted after retention period
  ✓ Automated deletion jobs
  ✓ Deletion logs maintained
  ✓ Compliance with local laws (GDPR, CCPA)

- User Rights
  ✓ Data access requests: Response within 30 days
  ✓ Deletion requests: Honor within 30 days
  ✓ Portability: Provide data in standard format
  ✓ Right to object: Honor within 10 days

- International Transfers
  ✓ Ensure legal basis for transfers
  ✓ Data Protection Agreements in place
  ✓ Compliance with local laws

Implementation:
- Privacy controls: Built into user dashboard
- Automation: Scheduled data deletion jobs
- Audit: Privacy audit trail
- Legal: Privacy policy reviewed by counsel
```

**SOC 2 Audit Process:**
```
Phase 1: Planning (Month 1)
- Identify systems in scope
- Map controls to criteria
- Determine test procedures
- Estimate effort

Phase 2: Implementation (Months 2-4)
- Document all controls
- Implement missing controls
- Test controls for effectiveness
- Remediate gaps

Phase 3: Observation Period (Months 5-8)
- Auditor observes controls
- Tests transactions
- Verifies effectiveness
- Notes any control failures

Phase 4: Reporting (Month 9)
- Auditor issues SOC 2 report
- Report valid for 1 year
- Cost: $30K-80K

Maintenance:
- Annual audit renewal
- Quarterly internal reviews
- Continuous monitoring
```

---

## Framework 2: PCI DSS 4.0 Compliance

**What is PCI DSS?**
PCI DSS (Payment Card Industry Data Security Standard) prevents credit card theft. Applies to any business handling card data. Non-compliance = $100K-$10M fines.

**Note:** PCI DSS 4.0 requirements took effect March 2025.

**Key Requirements:**

### Security Controls
```
1. Firewalls & Network Segmentation
   ✓ Firewalls between cardholder data environment (CDE) and internet
   ✓ Network segmentation: CDE isolated from other systems
   ✓ Inbound/outbound rules defined
   ✓ Default deny unless explicitly allowed

2. Change from Default Passwords
   ✓ No default credentials in production
   ✓ Unique passwords for all systems
   ✓ Passwords > 12 characters, complex
   ✓ Quarterly password changes

3. Malware Protection
   ✓ Antivirus on all systems touching card data
   ✓ Definitions updated weekly
   ✓ Log retention: 6 months
   ✓ Antivirus enabled at all times

4. Logging & Monitoring
   ✓ All access to cardholder data logged
   ✓ Real-time monitoring for anomalies
   ✓ Logs retained for 1 year
   ✓ Log centralization (can't delete if server compromised)

5. Access Control
   ✓ Only needed staff access card data
   ✓ Role-based access control
   ✓ Access denied by default
   ✓ Immediate termination revokes access

6. Encryption
   ✓ Card data encrypted in transit (TLS 1.2+)
   ✓ Card data encrypted at rest (AES-256)
   ✓ Never store: CVV, track data, PIN
   ✓ Always tokenize: Use token, not card number
```

### Card Data Handling
```
Never store:
✗ Full card number + CVV + expiry (even encrypted)
✗ Magnetic stripe data
✗ PIN
✗ Authentication code

Always store:
✓ Tokenized payment (Stripe token, not card)
✓ Last 4 digits (for receipt)
✓ Expiry month/year (for verification)
✓ Payment method (Visa, Mastercard)

Example flow:
1. Customer enters card: 4532-1111-2222-3333
2. JavaScript encryption on client
3. Token sent to Stripe: "stripe_token_xyz123"
4. Ecommerce stores token
5. Never see full card number
6. Use token for future charges
7. Ecommerce achieves PCI Level 1 compliance
```

### Compliance Levels
```
Level 1: >6M transactions/year
- Requirement: Annual onsite audit
- Cost: $80K-150K
- Frequency: Yearly

Level 2: 1-6M transactions/year
- Requirement: Attestation of Compliance (AOC)
- Cost: $10K-30K
- Frequency: Annually

Level 3: < 1M transactions/year
- Requirement: Self-Assessment Questionnaire (SAQ)
- Cost: $0 (self-assessment)
- Frequency: Annually

Level 4: < 1M transactions/year, all card-not-present
- Requirement: Self-Assessment Questionnaire
- Easiest if using hosted payment (Stripe, PayPal)
- Cost: $0
```

---

## Framework 3: Data Privacy & Localization

**GDPR (General Data Protection Regulation - Europe)**
```
Applies to: Any business with EU customers
Violation fine: €20M or 4% revenue (whichever higher)

Key requirements:
- Data residency: EU customer data stays in EU
- Consent: Explicit consent before processing
- Right to deletion: "Right to be forgotten"
- Right to access: User can download their data
- Breach notification: Notify within 72 hours
- Data Protection Officer: Some businesses need DPO

Implementation:
- Infrastructure: EU-only data centers
- Consent: Opt-in checkboxes (not pre-ticked)
- Deletion: Automated deletion jobs after retention period
- Breaches: Documented incident response plan
```

**CCPA (California Consumer Privacy Act)**
```
Applies to: California residents
Violation fine: $100-7,500 per incident

Key requirements:
- Notice: Privacy policy discloses data use
- Right to access: User can request data
- Right to delete: User can request deletion
- Right to opt-out: Can opt-out of data selling
- Non-discrimination: Can't penalize for exercising rights

Implementation:
- Privacy page: Include CCPA disclosures
- User dashboard: Allow data access/deletion
- Retention: Delete data after user requests
- Opt-out: Honor within 45 days
```

**LGPD (Lei Geral de Proteção de Dados - Brazil)**
```
Applies to: Brazilian residents
Violation fine: R$50M (≈$10M USD)

Similar to GDPR:
- Data residency: Brazilian data stays in Brazil
- Consent: Explicit consent required
- Deletion rights: "Right to be forgotten"
- Breach notification: Notify within 10 days
```

**Regional Implementation:**
```
User in Spain (GDPR):
✓ Data stored in EU data center (Ireland)
✓ Consent: Explicit opt-in
✓ Deletion: 30-day deletion on request
✓ Privacy officer required (maybe)

User in California (CCPA):
✓ Can access data via dashboard
✓ Can delete account + all data
✓ Can opt-out of data sales
✓ No penalty for exercising rights

User in Brazil (LGPD):
✓ Data stored in South America region
✓ Similar to GDPR requirements
✓ Explicit consent required
```

---

## Framework 4: Incident Response & Breach Protocol

**Incident Severity Levels:**

```
Level 1 - CRITICAL (> 1 million records affected)
- Immediate: Page CEO + Legal
- Action: Activate full incident response
- Communication: Notify customers within 24 hours
- Timeline: Begin investigation immediately
- Disclosure: Public notification may be required

Level 2 - HIGH (10K-1M records affected)
- Immediate: Page on-call security team
- Action: Investigate root cause
- Communication: Notify affected customers within 72 hours
- Timeline: Full forensics within 48 hours

Level 3 - MEDIUM (100-10K records affected)
- Response: Alert security team
- Investigation: Begin within 24 hours
- Communication: Notify customers if PII exposed
- Timeline: Resolution within 1 week

Level 4 - LOW (< 100 records or non-sensitive data)
- Response: Log and monitor
- Investigation: Complete within 72 hours
- Communication: No customer notification needed
- Timeline: Fix within 1 month
```

**Incident Response Plan:**

```
Step 1: Detect (Automated alerts)
- Alert fires: "1000 errors per minute"
- Alert fires: "Unauthorized API access"
- Alert fires: "Database exfiltration detected"
- Alert fires: "DDoS attack detected"

Step 2: Respond (First 15 minutes)
- On-call engineer acknowledges: "I'm investigating"
- Assess: What happened?
- Scope: How many customers affected?
- Impact: Is data at risk?
- Severity: Level 1, 2, 3, or 4?

Step 3: Contain (First hour)
- Stop active threats: Block attacker IPs, revoke tokens
- Preserve evidence: Capture logs, memory dumps
- Quarantine affected systems: Isolate, don't reboot
- Backup data: Ensure data available for recovery

Step 4: Investigate (First 24 hours)
- Root cause analysis: What caused this?
- Forensics: Who accessed what?
- Timeline: When did attack start?
- Impact: Exactly what data was exposed?

Step 5: Communicate (24-72 hours)
- Internal: Board, legal, executive team
- Customers: Email, call, in-app notification
- Media: If public breach, prepare statement
- Regulators: If required by law (GDPR = 72 hours)

Step 6: Remediate (First week)
- Fix: Address root cause (patch, configuration change)
- Deploy: Fix to production
- Verify: Confirm fix works
- Monitor: Watch for recurrence

Step 7: Improve (Post-incident)
- Blameless postmortem: What happened, why, lessons
- Action items: Prevent recurrence
- Follow-up: Implement safeguards within 30 days
- Communication: Inform customers of improvements

Timeline targets:
- Detection: < 5 minutes
- Response: < 15 minutes
- Containment: < 1 hour
- Investigation: < 24 hours
- Communication: < 72 hours (GDPR requirement)
- Remediation: < 1 week
- Post-incident review: < 1 week
```

**Audit Trail Preservation:**

```
Logs to maintain (1 year):
- Authentication logs: All logins, MFA challenges
- Authorization logs: All access to sensitive data
- Change logs: All code deployments, configuration changes
- Payment logs: All transactions (PCI requirement)
- API logs: All API calls with parameters (anonymized)
- Error logs: All system errors
- Security logs: Firewall, WAF, DDoS events

Storage:
- Centralized logging (ELK Stack, Splunk)
- Immutable storage (AWS S3 with versioning, can't delete)
- Retention: 1 year online, 7 years archived
- Encryption: At rest and in transit
- Access: Restricted to security and legal
- Audit: Audit trail of who accessed logs

Cost: $200-500/month for log retention
```

---

## Framework 5: API Security

**OAuth 2.0 for Authorization**

```
Never use API keys for user authentication
Instead use OAuth 2.0:

Flow:
1. User clicks "Login with Google"
2. Redirected to Google login
3. User enters Google password
4. Google asks: "Allow ecommerce access?"
5. User approves
6. Ecommerce receives: OAuth token (not password)
7. Ecommerce stores token
8. Ecommerce uses token for API calls
9. Google validates token

Benefits:
- User password never shared with ecommerce
- User controls permissions
- Permission revocation easy
- Social login reduces friction
```

**API Rate Limiting**

```
Prevent abuse: 100 requests/minute per IP

Limits:
- Anonymous user: 100 req/min
- Authenticated user: 1000 req/min
- Partner API key: 10K req/min
- Enterprise: Unlimited or custom

Response (if exceeded):
{
  "error": "rate_limit_exceeded",
  "retry_after": 60,
  "limit": 1000,
  "used": 1005
}

Header: Retry-After: 60 (wait 60 seconds)

Implementation:
- Redis: Track request counts by IP
- Reset: Every minute
- Alert: Alert if unusual patterns (distributed attack)
```

**DDoS Protection**

```
Layer 1: Network (Cloudflare, Akamai)
- Absorb volumetric attacks
- Filter at edge (millions of requests/sec)
- Cost: $200-5000/month

Layer 2: Application (Rate limiting)
- API rate limits
- Captcha on suspicious traffic
- Bot detection

Layer 3: Infrastructure
- Auto-scaling: Handle traffic spikes
- Load balancers: Distribute across servers
```

**API Security Best Practices**

```
1. HTTPS/TLS 1.2+ (Always)
   - No http:// (force https)
   - Certificates valid and non-expired
   - Certificate pinning (mobile apps)

2. Input Validation
   ✗ Accept: {"email": "test@example.com'; DROP TABLE users; --"}
   ✓ Validate: Reject invalid characters
   ✓ Sanitize: Remove dangerous input

3. Output Encoding
   - JSON: Encode special characters
   - HTML: Escape <, >, &
   - SQL: Use parameterized queries (never string concatenation)

4. Error Handling
   - Don't expose: Database errors, file paths, stack traces
   - Generic: "Invalid request" instead of "Column 'username' not found"
   - Logging: Log detailed errors for debugging

5. Secrets Management
   - Never commit: API keys, database passwords
   - Use: Environment variables or secret vault
   - Rotation: Every 90 days

6. CORS (Cross-Origin Resource Sharing)
   - Whitelist: Only allow trusted domains
   - Never use: * (allow all)
   - Specific: https://web.example.com only
```

---

**Guide Version:** 1.0 | **Status:** Production Compliance Frameworks | **Scope:** Enterprise-grade

