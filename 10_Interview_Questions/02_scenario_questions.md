# Scenario-Based Questions for System Design Interviews

This document contains "What would you do if..." scenario questions that test your ability to apply system design principles to real-world situations. These questions evaluate problem-solving skills, experience, and practical judgment.

---

## Table of Contents

1. [Handling Traffic Spikes](#1-handling-traffic-spikes)
2. [Database Performance Degradation](#2-database-performance-degradation)
3. [Service Outage Response](#3-service-outage-response)
4. [Data Corruption Discovery](#4-data-corruption-discovery)
5. [Third-Party API Failures](#5-third-party-api-failures)
6. [Scaling Bottlenecks](#6-scaling-bottlenecks)
7. [Security Breach Response](#7-security-breach-response)
8. [Migration Challenges](#8-migration-challenges)
9. [Cost Optimization](#9-cost-optimization)
10. [Performance Debugging](#10-performance-debugging)
11. [Deployment Failures](#11-deployment-failures)
12. [Data Consistency Issues](#12-data-consistency-issues)
13. [Capacity Planning](#13-capacity-planning)
14. [Feature Flag Incidents](#14-feature-flag-incidents)
15. [Cache Failures](#15-cache-failures)

---

## 1. Handling Traffic Spikes

### Question
**"What would you do if your system suddenly received 10x normal traffic due to a viral event?"**

### Sample Answer

**Immediate Response (First 5 minutes):**

1. **Assess the situation:**
   - Check monitoring dashboards for error rates, latency, and resource utilization
   - Identify which services are affected
   - Determine if this is legitimate traffic or an attack

2. **Enable emergency measures:**
   - Activate auto-scaling if not already enabled
   - Enable rate limiting to protect critical services
   - Activate circuit breakers to prevent cascade failures

3. **Shed non-critical load:**
   - Disable non-essential features (recommendations, analytics)
   - Return cached responses where possible
   - Enable graceful degradation modes

**Short-term Actions (First hour):**

4. **Scale infrastructure:**
   - Manually scale up database read replicas
   - Increase cache capacity
   - Add more application server instances
   - Consider CDN cache TTL increases

5. **Optimize for the traffic pattern:**
   - Identify hot paths and optimize queries
   - Increase connection pool sizes
   - Adjust timeout configurations

**Communication:**
- Alert stakeholders about the situation
- Prepare status page updates
- Coordinate with support teams

**Post-incident:**
- Conduct retrospective on what worked/didn't
- Implement permanent capacity improvements
- Update runbooks for future events
- Consider implementing predictive scaling

### What Interviewers Are Looking For
- Calm, structured approach to crisis management
- Prioritization skills (protect core functionality first)
- Knowledge of various scaling and protection mechanisms
- Communication awareness
- Learning mindset (post-incident improvement)

### Tips for Answering
- Start with assessment before taking action
- Show you understand trade-offs (e.g., graceful degradation)
- Mention monitoring and observability
- Include communication with stakeholders
- Discuss prevention for next time

### Common Follow-Ups
- "How would you distinguish between a DDoS attack and legitimate traffic?"
- "What metrics would you look at first?"
- "How would you decide which features to disable?"
- "What if auto-scaling isn't fast enough?"

---

## 2. Database Performance Degradation

### Question
**"What would you do if you noticed your database queries are running 5x slower than usual?"**

### Sample Answer

**Immediate Investigation:**

1. **Check recent changes:**
   - Any new deployments in the last 24 hours?
   - Configuration changes?
   - New features that might have introduced new queries?

2. **Analyze database metrics:**
   - CPU, memory, disk I/O utilization
   - Connection count and wait times
   - Lock contention and deadlocks
   - Replication lag (if applicable)

3. **Identify problematic queries:**
   - Enable slow query logging
   - Check query execution plans (EXPLAIN)
   - Look for missing indexes
   - Identify N+1 query patterns

**Common Root Causes and Solutions:**

**Missing Index:**
```sql
-- Identify slow queries
SELECT * FROM pg_stat_statements ORDER BY total_time DESC LIMIT 10;

-- Check if index would help
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123;

-- Add appropriate index
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
```

**Lock Contention:**
- Identify blocking queries
- Optimize long-running transactions
- Consider read replicas for read traffic

**Resource Exhaustion:**
- Scale up database instance
- Optimize connection pooling
- Implement query result caching

**Data Growth:**
- Archive old data
- Implement partitioning
- Consider sharding for long-term

**Immediate Mitigations:**
- Route read traffic to replicas
- Enable query result caching
- Kill long-running queries if safe
- Implement query timeouts

**Prevention:**
- Regular query performance reviews
- Automated slow query alerts
- Query analysis in CI/CD pipeline
- Regular index maintenance

### What Interviewers Are Looking For
- Systematic debugging approach
- Knowledge of database performance tools
- Understanding of common performance issues
- Balance between immediate fixes and root cause resolution

### Tips for Answering
- Don't jump to solutions without investigation
- Mention specific tools (EXPLAIN, slow query logs)
- Show awareness of the impact of changes (CONCURRENTLY for indexes)
- Include prevention measures

### Common Follow-Ups
- "How would you add an index without downtime?"
- "What if the problem is with your ORM?"
- "How would you handle this if it's a peak traffic period?"
- "What monitoring would you add to catch this earlier?"

---

## 3. Service Outage Response

### Question
**"What would you do if one of your critical microservices went down in production?"**

### Sample Answer

**Incident Response Framework:**

**1. Detection and Triage (0-5 minutes):**
```
- Verify the outage (is it real or monitoring issue?)
- Assess blast radius (what's affected?)
- Determine severity level
- Initiate incident response if needed
```

**2. Immediate Containment:**

**If the service can be restarted:**
- Attempt pod/container restart
- Check for resource limits (OOM kills)
- Review recent deployments

**If restart doesn't help:**
- Check dependencies (database, cache, external services)
- Review error logs for stack traces
- Check infrastructure (nodes, network)

**3. Enable Fallback Mechanisms:**
```
Service A (down) -> Circuit breaker opens -> Fallback to:
  - Cached responses
  - Degraded functionality
  - Graceful error messages
```

**4. Communication:**
- Update status page
- Notify affected teams
- Keep stakeholders informed with regular updates

**5. Resolution Steps:**

**If caused by bad deployment:**
- Rollback to previous version
- Verify service recovery
- Investigate what went wrong

**If caused by dependency:**
- Fail over to backup if available
- Contact dependency team/provider
- Implement workarounds

**If caused by infrastructure:**
- Check cloud provider status
- Fail over to another region/zone
- Scale replacement capacity

**6. Post-Incident:**
- Document timeline and actions taken
- Conduct blameless post-mortem
- Identify improvements
- Update runbooks and alerts

### Sample Incident Timeline

| Time | Action |
|------|--------|
| 0:00 | Alert fires, on-call engineer paged |
| 0:02 | Engineer acknowledges, begins investigation |
| 0:05 | Root cause identified: OOM due to memory leak |
| 0:08 | Decision to rollback |
| 0:12 | Rollback complete, service recovering |
| 0:15 | Service fully recovered, monitoring |
| 0:30 | Incident closed, post-mortem scheduled |

### What Interviewers Are Looking For
- Structured incident response process
- Prioritization (containment over root cause)
- Knowledge of fallback mechanisms
- Communication awareness
- Learning from incidents

### Tips for Answering
- Emphasize reducing impact first, then finding root cause
- Mention specific rollback strategies
- Include communication throughout
- Show you value blameless post-mortems

### Common Follow-Ups
- "How do you decide between rollback and fix-forward?"
- "What if you can't identify the root cause quickly?"
- "How do you handle cascading failures?"
- "Who do you communicate with and when?"

---

## 4. Data Corruption Discovery

### Question
**"What would you do if you discovered that some user data has been corrupted or incorrectly modified?"**

### Sample Answer

**Immediate Actions:**

**1. Stop the Bleeding:**
- Identify the source of corruption (is it ongoing?)
- If ongoing, disable the feature/service causing it
- Prevent further writes to affected data if possible

**2. Assess the Damage:**
```
Questions to answer:
- How many records are affected?
- Which time period?
- What type of corruption (wrong values, missing data, duplicates)?
- Can we identify all affected records?
```

**3. Preserve Evidence:**
- Take database snapshots before any fixes
- Export affected records for analysis
- Collect relevant logs

**Investigation:**

**4. Root Cause Analysis:**
```
Common causes:
- Bug in application code
- Race condition
- Failed migration
- Manual intervention gone wrong
- Third-party integration issue
```

**5. Identify Scope:**
```sql
-- Example: Find all orders affected by corruption
SELECT COUNT(*), MIN(created_at), MAX(created_at)
FROM orders
WHERE total_amount < 0  -- Corruption indicator
  OR customer_id IS NULL;
```

**Recovery Strategy:**

**Option 1: Restore from Backup**
- Best for extensive corruption
- May lose recent good data
- Point-in-time recovery if available

**Option 2: Replay from Event Log**
- If using event sourcing
- Rebuild state from events
- Skip corrupted events

**Option 3: Manual/Script Fix**
```sql
-- Only if corruption is well-understood and limited
UPDATE orders
SET total_amount = (
  SELECT SUM(price * quantity)
  FROM order_items
  WHERE order_id = orders.id
)
WHERE order_id IN (SELECT id FROM corrupted_order_ids);
```

**Option 4: User Notification and Manual Correction**
- When automated fix is risky
- Provide tools for users to verify/correct

**Communication:**
- Inform affected users if necessary
- Regulatory notification if required (GDPR, etc.)
- Internal stakeholders and leadership

**Prevention:**
- Add data validation at multiple layers
- Implement audit logging
- Regular data integrity checks
- Database constraints
- Better testing of data migrations

### What Interviewers Are Looking For
- Calm approach to crisis
- Data preservation instinct
- Systematic investigation process
- Knowledge of recovery options
- Awareness of compliance requirements

### Tips for Answering
- Emphasize stopping ongoing corruption first
- Mention preserving evidence/snapshots
- Show multiple recovery strategies
- Include user communication considerations
- Discuss prevention measures

### Common Follow-Ups
- "How would you prevent this in the future?"
- "What if you can't identify all affected records?"
- "How do you handle this if there's regulatory impact?"
- "What if the backup is also corrupted?"

---

## 5. Third-Party API Failures

### Question
**"What would you do if a critical third-party API (e.g., payment processor) started failing?"**

### Sample Answer

**Immediate Response:**

**1. Verify the Issue:**
- Is it our integration or the provider?
- Check provider status page
- Try manual API call to confirm

**2. Activate Circuit Breaker:**
```
Circuit breaker pattern:
- Closed: Normal operation
- Open: Fail fast, return fallback
- Half-Open: Test if service recovered
```

**3. Implement Fallback Strategies:**

**For Payment Processing:**
```
Primary: Stripe -> Failing
Fallbacks:
1. Secondary provider (PayPal, Braintree)
2. Queue for later retry
3. Manual processing for high-value transactions
4. Graceful error with retry option for users
```

**Fallback Implementation:**
```python
async def process_payment(payment):
    try:
        return await primary_processor.charge(payment)
    except CircuitBreakerOpen:
        if payment.amount > MANUAL_THRESHOLD:
            return await queue_for_manual_review(payment)
        else:
            return await fallback_processor.charge(payment)
    except ProviderError:
        circuit_breaker.record_failure()
        return await retry_with_backoff(payment)
```

**4. Communication:**
- Internal alert to engineering/support
- Update status page if user-facing
- Prepare user-facing messaging

**Retry Strategy:**
```
Exponential backoff with jitter:
- 1st retry: 1s + random(0-500ms)
- 2nd retry: 2s + random(0-500ms)
- 3rd retry: 4s + random(0-500ms)
- Max retries: 5
- Dead letter queue for failed retries
```

**5. Queue for Later Processing:**
- Store failed requests in durable queue
- Process when service recovers
- Ensure idempotency for retries

**Long-term Mitigations:**
- Multi-provider strategy
- Timeouts and circuit breakers on all external calls
- Regular provider health monitoring
- Contract testing for integrations
- SLA monitoring and alerting

### Sample Architecture for Resilience

```
User Request
    |
    v
[API Gateway] -> [Rate Limit] -> [Circuit Breaker]
    |                                    |
    v                                    v
[Primary Provider] -- fail --> [Fallback Provider]
    |                                    |
    v                                    v
[Response] <---- success --------------- +
    |
    v (if all fail)
[Queue for Retry] -> [Dead Letter Queue] -> [Manual Review]
```

### What Interviewers Are Looking For
- Understanding of resilience patterns
- Fallback strategy thinking
- Knowledge of retry best practices
- User experience consideration
- Proactive prevention measures

### Tips for Answering
- Start with verification (is it really failing?)
- Show multiple fallback layers
- Mention specific patterns (circuit breaker, retry with backoff)
- Include user experience considerations
- Discuss prevention and monitoring

### Common Follow-Ups
- "How do you test your fallback mechanisms?"
- "What if the secondary provider also fails?"
- "How do you handle transactions that were partially processed?"
- "How do you decide timeout values?"

---

## 6. Scaling Bottlenecks

### Question
**"What would you do if you identified that a single service is the bottleneck preventing your system from scaling?"**

### Sample Answer

**Investigation Phase:**

**1. Profile the Bottleneck:**
```
Questions to answer:
- Is it CPU-bound, I/O-bound, or memory-bound?
- What's the resource utilization at current load?
- What's the throughput per instance?
- Where does time go in request processing?
```

**2. Identify Root Cause:**

**CPU-bound:**
- Expensive computations
- Inefficient algorithms
- JSON serialization/parsing
- Compression

**I/O-bound:**
- Database queries
- External API calls
- File system operations
- Network latency

**Memory-bound:**
- Large object allocation
- Memory leaks
- Cache sizes
- Connection pools

**3. Optimization Strategies:**

**Vertical Scaling (Quick Win):**
```
Increase instance size:
- More CPU cores
- More memory
- Faster I/O (SSD, enhanced networking)
```

**Horizontal Scaling:**
```
If service is stateless:
- Add more instances
- Implement load balancing
- Auto-scaling policies

If service has state:
- Extract state to external store
- Implement sharding
- Consider stateless redesign
```

**Code Optimization:**
```python
# Before: N+1 queries
for order in orders:
    items = db.query(f"SELECT * FROM items WHERE order_id = {order.id}")

# After: Single query with join
orders_with_items = db.query("""
    SELECT o.*, i.*
    FROM orders o
    JOIN items i ON o.id = i.order_id
    WHERE o.id IN (...)
""")
```

**Caching:**
```
Add caching at appropriate levels:
- Application cache (in-memory)
- Distributed cache (Redis)
- CDN for static content
- Database query cache
```

**Async Processing:**
```
Move expensive operations off the critical path:
- Queue heavy computations
- Background jobs for non-urgent work
- Event-driven for downstream processing
```

**4. Architecture Changes (Longer-term):**

```
Decomposition options:
- Split service by functionality
- Separate read and write paths (CQRS)
- Move to event-driven architecture
- Implement database read replicas
```

### Decision Matrix

| Bottleneck Type | Quick Fix | Medium-term | Long-term |
|----------------|-----------|-------------|-----------|
| CPU-bound | Vertical scale | Algorithm optimization | Decompose service |
| I/O-bound | Add caching | Async processing | Database sharding |
| Memory-bound | Increase memory | Fix leaks, optimize | Redesign data model |
| Network-bound | Connection pooling | CDN, edge caching | Regional deployment |

### What Interviewers Are Looking For
- Systematic approach to performance analysis
- Knowledge of various optimization techniques
- Understanding of scaling strategies
- Ability to balance quick fixes vs long-term solutions

### Tips for Answering
- Start with profiling/measurement
- Discuss both quick wins and strategic fixes
- Show understanding of different bottleneck types
- Include trade-offs of each approach

### Common Follow-Ups
- "How do you decide between optimizing code vs adding more hardware?"
- "What profiling tools would you use?"
- "How do you handle a bottleneck in a database?"
- "What if the bottleneck is in a third-party service?"

---

## 7. Security Breach Response

### Question
**"What would you do if you discovered unauthorized access to your system or a potential security breach?"**

### Sample Answer

**Immediate Response (First 15 minutes):**

**1. Verify and Assess:**
- Confirm the breach is real (not a false positive)
- Determine scope: What was accessed?
- Identify attack vector: How did they get in?

**2. Contain the Threat:**
```
Containment actions:
- Revoke compromised credentials
- Block suspicious IP addresses
- Disable compromised accounts
- Isolate affected systems
- Preserve logs and evidence
```

**3. Activate Incident Response:**
- Page security team
- Engage incident commander
- Start incident documentation

**Short-term Response (First few hours):**

**4. Investigation:**
```
Forensic analysis:
- Review access logs
- Check for lateral movement
- Identify data exfiltration
- Timeline of attacker activity
- Identify all affected systems
```

**5. Communication:**

**Internal:**
- Executive leadership
- Legal team
- PR/Communications
- Affected teams

**External (if required):**
- Law enforcement
- Regulators (GDPR: 72 hours)
- Affected customers
- Security researchers (if they reported)

**6. Eradication:**
```
Remove attacker access:
- Patch vulnerability exploited
- Rotate all credentials
- Remove backdoors
- Update firewall rules
- Deploy security patches
```

**7. Recovery:**
- Restore from clean backups if needed
- Gradually restore services
- Monitor for re-entry attempts
- Verify system integrity

**Post-Incident:**

**8. Documentation and Learning:**
- Complete incident report
- Timeline of events
- Actions taken
- Recommendations

**9. Improvements:**
```
Common improvements:
- Additional monitoring/alerting
- Penetration testing
- Security training
- Access control review
- Vulnerability scanning
```

### Incident Response Checklist

| Phase | Actions | Owner |
|-------|---------|-------|
| Detection | Verify alert, assess severity | On-call |
| Containment | Block access, preserve evidence | Security |
| Investigation | Forensics, scope assessment | Security |
| Communication | Internal/external notification | Leadership |
| Eradication | Remove threat, patch vulnerabilities | Engineering |
| Recovery | Restore services, verify | Engineering |
| Post-mortem | Document, improve | All |

### What Interviewers Are Looking For
- Structured incident response approach
- Evidence preservation awareness
- Understanding of legal/compliance requirements
- Communication prioritization
- Learning from incidents

### Tips for Answering
- Emphasize containment before investigation
- Mention evidence preservation
- Include legal/compliance considerations
- Show awareness of communication needs
- Focus on prevention improvements

### Common Follow-Ups
- "When would you involve law enforcement?"
- "How do you balance investigation with business continuity?"
- "What if you're not sure the breach is contained?"
- "How do you communicate with affected customers?"

---

## 8. Migration Challenges

### Question
**"What would you do if a database migration is taking much longer than expected and blocking deployments?"**

### Sample Answer

**Immediate Assessment:**

**1. Understand the Situation:**
```
Questions to answer:
- How long has it been running?
- What's the expected completion time?
- Is it blocking critical deployments?
- Is it causing production issues?
```

**2. Analyze the Migration:**
```sql
-- Check migration progress
SELECT phase, status, running_time
FROM migration_status;

-- Check what's blocking
SELECT * FROM pg_stat_activity
WHERE state != 'idle'
AND query LIKE '%migration%';

-- Check lock contention
SELECT * FROM pg_locks WHERE NOT granted;
```

**Decision Framework:**

**Option 1: Let It Complete**
- If close to completion
- No critical impact
- Monitor closely

**Option 2: Optimize In Progress**
```
Potential optimizations:
- Increase batch sizes
- Adjust checkpoint settings
- Disable triggers temporarily
- Reduce concurrent operations
```

**Option 3: Cancel and Redesign**
- If far from completion
- Causing significant issues
- Can safely rollback

**3. Future Migration Strategies:**

**Online/Live Migrations:**
```
Step 1: Add new column (nullable)
Step 2: Backfill data in batches
Step 3: Deploy code to write to both
Step 4: Verify data consistency
Step 5: Make new column required
Step 6: Remove old column
```

**Expand-Contract Pattern:**
```
Expand: Add new structure alongside old
Migrate: Move data in small batches
Contract: Remove old structure after verification
```

**Ghost/pt-online-schema-change:**
```
- Creates shadow table
- Applies changes to shadow
- Syncs data via triggers
- Atomic table swap
- Minimal locking
```

**Batch Processing:**
```python
def migrate_in_batches(table, batch_size=1000):
    last_id = 0
    while True:
        rows = db.query(f"""
            SELECT * FROM {table}
            WHERE id > {last_id}
            ORDER BY id LIMIT {batch_size}
        """)
        if not rows:
            break

        process_batch(rows)
        last_id = rows[-1].id

        # Sleep between batches to reduce load
        time.sleep(0.1)
```

### Migration Best Practices

| Practice | Benefit |
|----------|---------|
| Test on production-size data | Accurate time estimates |
| Run during low-traffic periods | Less impact |
| Make migrations reversible | Easy rollback |
| Use feature flags | Decouple deployment from migration |
| Monitor during migration | Early problem detection |
| Have rollback plan | Quick recovery |

### What Interviewers Are Looking For
- Ability to assess risk vs reward
- Knowledge of migration techniques
- Understanding of database operations
- Experience with large-scale migrations

### Tips for Answering
- Start with assessment, not action
- Discuss multiple options
- Show knowledge of online migration techniques
- Include prevention for future migrations

### Common Follow-Ups
- "How do you estimate migration time?"
- "What if the migration is irreversible?"
- "How do you handle migrations with zero downtime?"
- "What if data validation fails after migration?"

---

## 9. Cost Optimization

### Question
**"What would you do if your cloud infrastructure costs suddenly increased by 50%?"**

### Sample Answer

**Investigation Phase:**

**1. Identify Cost Drivers:**
```
Analyze cloud billing:
- Which services increased most?
- Which accounts/projects?
- What time period?
- New resources or usage increase?
```

**2. Common Cost Increase Causes:**

**Traffic/Usage Growth:**
- Legitimate business growth
- Viral event
- DDoS attack
- Inefficient code deployment

**Resource Sprawl:**
- Orphaned resources
- Test environments not cleaned up
- Over-provisioned instances
- Unused storage

**Configuration Changes:**
- Auto-scaling without limits
- Expensive instance types
- Multi-AZ/region accidentally enabled
- Backup retention extended

**3. Quick Wins:**

**Identify Waste:**
```
Look for:
- Unattached EBS volumes
- Unused Elastic IPs
- Idle load balancers
- Old snapshots
- Stopped but provisioned resources
```

**Right-size Resources:**
```
Analysis approach:
- CPU utilization < 20%? Downsize
- Memory utilization < 30%? Downsize
- Consider burstable instances (t3, etc.)
```

**4. Medium-term Optimizations:**

**Reserved Instances/Savings Plans:**
```
For predictable workloads:
- 1-year RI: ~40% savings
- 3-year RI: ~60% savings
- Savings Plans for flexibility
```

**Spot Instances:**
```
For fault-tolerant workloads:
- Up to 90% savings
- Good for: batch processing, CI/CD, dev/test
- Use spot fleet for availability
```

**Architecture Changes:**
```
Cost-efficient patterns:
- Serverless for variable workloads
- Auto-scaling with appropriate bounds
- Caching to reduce compute/database
- CDN for static content
- Data tiering (hot/cold storage)
```

**5. Governance:**

```
Implement cost controls:
- Budget alerts
- Tagging requirements
- Auto-shutdown for dev environments
- Regular cost reviews
- FinOps practices
```

### Cost Optimization Checklist

| Area | Action | Savings Potential |
|------|--------|------------------|
| Compute | Right-sizing | 20-40% |
| Compute | Reserved/Spot | 40-90% |
| Storage | Lifecycle policies | 30-50% |
| Database | Reserved instances | 40-60% |
| Network | CDN, caching | 20-40% |
| Waste | Orphaned resources | 10-20% |

### What Interviewers Are Looking For
- Systematic approach to cost analysis
- Knowledge of cloud pricing models
- Balance between cost and performance
- Long-term sustainability thinking

### Tips for Answering
- Start with investigation, not cutting
- Show knowledge of cloud pricing
- Discuss both quick wins and strategic changes
- Include governance/prevention

### Common Follow-Ups
- "How do you balance cost vs performance?"
- "How would you implement cost allocation?"
- "What's your approach to reserved instances vs on-demand?"
- "How do you handle cost optimization across teams?"

---

## 10. Performance Debugging

### Question
**"What would you do if users report that the application is slow, but your monitoring shows everything is fine?"**

### Sample Answer

**The Gap Between Monitoring and User Experience:**

**1. Validate User Reports:**
```
Gather details:
- Which pages/features are slow?
- What geography are users in?
- What devices/browsers?
- What time of day?
- Can we reproduce?
```

**2. Check for Blind Spots:**

**Client-Side Performance:**
```javascript
// What monitoring might miss:
- JavaScript execution time
- DOM rendering
- Third-party scripts
- Web fonts loading
- Client-side routing
- Service worker issues
```

**Edge Cases:**
```
Consider:
- Specific user segments (new users, power users)
- Geographic regions
- Network conditions (mobile, slow connections)
- Peak vs off-peak times
- Specific browsers/devices
```

**3. Expand Monitoring:**

**Real User Monitoring (RUM):**
```javascript
// Capture actual user experience
performance.measure('page-load', 'navigationStart', 'loadEventEnd');
performance.measure('time-to-interactive', 'navigationStart', 'firstInteractive');

// Report to analytics
reportPerformanceMetrics({
  ttfb: timing.responseStart - timing.requestStart,
  fcp: paintEntries.find(e => e.name === 'first-contentful-paint'),
  lcp: largestContentfulPaint,
  cls: cumulativeLayoutShift,
  fid: firstInputDelay
});
```

**Synthetic Monitoring:**
```
Add tests from:
- Multiple geographic locations
- Different network speeds
- Various devices
- Critical user journeys
```

**4. Common Hidden Causes:**

| Symptom | Possible Cause | Detection |
|---------|---------------|-----------|
| Slow initial load | Large JavaScript bundles | Bundle analysis |
| Slow interactions | Main thread blocking | Performance profiler |
| Intermittent slowness | Third-party scripts | Network waterfall |
| Slow for some users | CDN routing issues | Multi-region testing |
| Slow on mobile | Unoptimized images | Lighthouse audit |
| Slow after login | N+1 queries for user data | SQL logging |

**5. Investigation Tools:**

**Browser DevTools:**
```
- Performance tab for runtime analysis
- Network tab for request timing
- Lighthouse for audits
- Coverage for unused code
```

**Server-Side:**
```
- Distributed tracing (Jaeger, Zipkin)
- APM tools (New Relic, Datadog)
- Database query analysis
- Log correlation
```

**6. Quick Diagnostic Steps:**

```
1. Reproduce in incognito mode (no extensions)
2. Check Network tab waterfall
3. Look at Core Web Vitals
4. Test from different locations
5. Profile JavaScript execution
6. Analyze bundle sizes
7. Check for layout shifts
```

### What Interviewers Are Looking For
- Understanding that monitoring has blind spots
- Knowledge of client-side performance
- User-centric thinking
- Systematic debugging approach

### Tips for Answering
- Validate user reports with specific questions
- Distinguish server vs client performance
- Mention specific tools and metrics
- Show understanding of real user monitoring

### Common Follow-Ups
- "What Core Web Vitals would you monitor?"
- "How do you balance performance monitoring with privacy?"
- "How would you debug a mobile-specific performance issue?"
- "What if the problem only happens for a small percentage of users?"

---

## 11. Deployment Failures

### Question
**"What would you do if a deployment starts causing errors in production and rolling back also fails?"**

### Sample Answer

**Crisis Response:**

**1. Immediate Assessment:**
```
Understand the situation:
- What errors are occurring?
- What percentage of traffic is affected?
- What specifically failed in the rollback?
- Is the system degrading further?
```

**2. Containment Options:**

**Traffic Redirection:**
```
If possible:
- Route traffic to healthy instances
- Enable maintenance mode
- Geographic failover if available
```

**Partial Rollback:**
```
If full rollback fails:
- Rollback specific services
- Revert configuration separately from code
- Manual instance replacement
```

**3. Diagnose Rollback Failure:**

**Common Rollback Failures:**

| Failure Type | Cause | Mitigation |
|--------------|-------|------------|
| Database incompatibility | Migration not backward compatible | Restore database backup |
| Missing artifacts | Old artifacts expired | Rebuild from version control |
| Configuration drift | Config changed during deployment | Version configuration |
| Stateful dependencies | External system changed | Manual coordination |
| Infrastructure changes | IaC changed | Terraform rollback |

**4. Emergency Fixes:**

**Hot Fix Deployment:**
```
When rollback fails:
1. Identify minimum fix needed
2. Skip normal pipeline stages if necessary
3. Deploy directly to affected instances
4. Monitor closely
5. Follow up with proper fix
```

**Database Recovery:**
```sql
-- If migration broke data
-- Option 1: Point-in-time recovery
-- Option 2: Run reverse migration
-- Option 3: Manual data fix

-- Example: Restore dropped column
ALTER TABLE orders ADD COLUMN old_status VARCHAR(50);
UPDATE orders SET old_status = status;  -- If recoverable
```

**5. Communication:**
```
During incident:
- Internal: Engineering lead, on-call, leadership
- External: Status page updates, support teams

Communication template:
"[Severity] Production deployment issue
Impact: [affected functionality]
Status: [current actions]
ETA: [expected resolution time]
Next update: [time]"
```

**6. Prevention:**

**Better Deployment Practices:**
```
Implement:
- Canary deployments (1% -> 10% -> 50% -> 100%)
- Feature flags to disable new code
- Blue-green deployments
- Automated rollback triggers
- Database migration validation
- Artifact retention policies
```

**Rollback Testing:**
```
In CI/CD pipeline:
1. Deploy version N
2. Run smoke tests
3. Rollback to version N-1
4. Verify rollback works
5. Deploy version N again
```

### Emergency Runbook Checklist

| Step | Action | Owner |
|------|--------|-------|
| 1 | Assess impact and severity | On-call |
| 2 | Attempt standard rollback | On-call |
| 3 | If failed, enable maintenance mode | On-call |
| 4 | Escalate to deployment team | On-call |
| 5 | Diagnose rollback failure | Team |
| 6 | Execute emergency fix | Team |
| 7 | Verify system stability | Team |
| 8 | Conduct post-mortem | All |

### What Interviewers Are Looking For
- Calm under pressure
- Multiple fallback strategies
- Knowledge of deployment patterns
- Prevention-focused thinking

### Tips for Answering
- Show prioritization (user impact first)
- Discuss multiple options, not just one
- Include communication aspects
- Emphasize prevention for future

### Common Follow-Ups
- "How do you test rollbacks?"
- "What makes a migration 'backward compatible'?"
- "How do you decide between fix-forward and rollback?"
- "What deployment patterns reduce this risk?"

---

## 12. Data Consistency Issues

### Question
**"What would you do if you discovered that data between two microservices is out of sync?"**

### Sample Answer

**Investigation Phase:**

**1. Understand the Inconsistency:**
```
Questions to answer:
- Which services are out of sync?
- What data is inconsistent?
- How much data is affected?
- When did it start?
- What's the business impact?
```

**2. Identify Root Cause:**

**Common Causes:**

| Cause | Detection | Example |
|-------|-----------|---------|
| Event lost | Message queue metrics | Order created, inventory not updated |
| Processing failure | Error logs, DLQ | Event processed but operation failed |
| Race condition | Timing analysis | Concurrent updates overwriting |
| Schema mismatch | Event validation | New field not propagated |
| Dual writes | Code review | Writing to two places, one fails |

**3. Immediate Mitigation:**

**Option 1: Manual Reconciliation**
```python
# Compare data between services
def reconcile_orders_inventory():
    orders = orders_service.get_all_orders()
    inventory_records = inventory_service.get_all()

    for order in orders:
        expected_inventory = calculate_expected(order)
        actual_inventory = inventory_records.get(order.product_id)

        if expected_inventory != actual_inventory:
            discrepancies.append({
                'order_id': order.id,
                'expected': expected_inventory,
                'actual': actual_inventory
            })

    return discrepancies
```

**Option 2: Event Replay**
```
If event sourced:
1. Identify affected time range
2. Reset consumer position
3. Replay events from that point
4. Verify consistency
```

**Option 3: Source of Truth Sync**
```
Designate authoritative source:
1. Export from source of truth
2. Compare with replica
3. Update replica to match
```

**4. Prevention Strategies:**

**Saga Pattern for Distributed Transactions:**
```
Order Service:
1. Create order (PENDING)
2. Request inventory reservation
3. On success: Order CONFIRMED
4. On failure: Order CANCELLED (compensation)

Inventory Service:
1. Reserve inventory
2. If order cancelled: Release reservation (compensation)
```

**Outbox Pattern:**
```sql
-- Instead of dual writes, use transactional outbox
BEGIN TRANSACTION;
  INSERT INTO orders (id, ...) VALUES (...);
  INSERT INTO outbox (event_type, payload)
    VALUES ('OrderCreated', '{"order_id": ...}');
COMMIT;

-- Separate process reads outbox, publishes to message queue
```

**Event Sourcing:**
```
All state changes as events:
- Events are append-only
- State derived from events
- Can rebuild any point in time
- Natural audit trail
```

**5. Monitoring and Alerting:**

```python
# Periodic consistency checks
def check_consistency():
    orders_count = orders_db.count()
    inventory_reservations = inventory_db.count_reservations()

    if abs(orders_count - inventory_reservations) > THRESHOLD:
        alert("Data consistency issue detected",
              orders=orders_count,
              reservations=inventory_reservations)
```

### Consistency Strategies Comparison

| Strategy | Consistency | Complexity | Use Case |
|----------|-------------|------------|----------|
| Two-phase commit | Strong | High | Critical transactions |
| Saga | Eventual | Medium | Distributed transactions |
| Outbox pattern | At-least-once | Medium | Event publishing |
| Event sourcing | Complete audit | High | Audit requirements |
| Periodic reconciliation | Eventual | Low | Background correction |

### What Interviewers Are Looking For
- Understanding of distributed data challenges
- Knowledge of consistency patterns
- Practical reconciliation approaches
- Prevention-focused thinking

### Tips for Answering
- Start with understanding the scope
- Discuss multiple resolution strategies
- Show knowledge of patterns (saga, outbox)
- Include monitoring/prevention

### Common Follow-Ups
- "How do you choose between saga and 2PC?"
- "What is the outbox pattern?"
- "How do you handle compensating transactions?"
- "How often should you run reconciliation?"

---

## 13. Capacity Planning

### Question
**"What would you do if you learned that traffic to your system is expected to 10x in 6 months due to a new product launch?"**

### Sample Answer

**Assessment Phase:**

**1. Understand Current Capacity:**
```
Baseline metrics:
- Current peak traffic
- Resource utilization at peak
- Response time percentiles
- Error rates
- Database connections
- Storage growth rate
```

**2. Identify Bottlenecks:**
```
Common bottlenecks at scale:
- Database connections/queries
- API rate limits
- Network bandwidth
- Memory per instance
- Third-party service limits
```

**Planning Phase:**

**3. Capacity Model:**

```python
# Simple capacity model
def calculate_required_capacity(target_traffic, current_capacity):
    headroom = 1.3  # 30% buffer

    return {
        'web_servers': ceil(target_traffic / current_throughput_per_server * headroom),
        'database_connections': target_traffic * avg_queries_per_request * headroom,
        'cache_memory': current_cache_size * (target_traffic / current_traffic) * headroom,
        'storage': current_storage * growth_rate * 6 * headroom
    }
```

**4. Scaling Strategy by Component:**

| Component | Current | Strategy | Notes |
|-----------|---------|----------|-------|
| Web tier | 10 instances | Auto-scale to 100 | Horizontal scaling |
| Database writes | 1 primary | Add sharding | Plan shard key now |
| Database reads | 2 replicas | Add to 10 | Implement connection pooling |
| Cache | 3 Redis nodes | Scale to cluster | Implement consistent hashing |
| CDN | Basic | Enterprise tier | Increase cache capacity |
| Message queue | Standard | Increase partitions | Pre-partition topics |

**5. Implementation Roadmap:**

**Month 1-2: Foundation**
```
- Implement auto-scaling with appropriate bounds
- Add caching where missing
- Optimize critical database queries
- Set up load testing infrastructure
```

**Month 3-4: Architecture**
```
- Implement database read replicas
- Add connection pooling
- Prepare sharding strategy
- CDN optimization
```

**Month 5-6: Validation**
```
- Load test at 10x capacity
- Chaos engineering tests
- Runbook preparation
- War room planning
```

**6. Load Testing Plan:**

```yaml
# Load test progression
test_phases:
  - name: baseline
    target_rps: current_peak
    duration: 30m

  - name: 2x_growth
    target_rps: current_peak * 2
    duration: 1h

  - name: 5x_growth
    target_rps: current_peak * 5
    duration: 2h

  - name: 10x_target
    target_rps: current_peak * 10
    duration: 4h

  - name: 12x_buffer
    target_rps: current_peak * 12
    duration: 1h
```

**7. Risk Mitigation:**

```
Prepare for unknowns:
- Feature flags for instant disable
- Graceful degradation modes
- War room schedule for launch
- Vendor escalation contacts
- Rollback procedures
```

### Capacity Planning Checklist

| Area | Questions | Action Items |
|------|-----------|--------------|
| Compute | Can we auto-scale? Limits? | Configure auto-scaling |
| Database | Query performance at 10x? | Index optimization, sharding |
| Cache | Hit rate sustainable? | Cache warming, sizing |
| Network | Bandwidth limits? | CDN, regional deployment |
| Third-party | Rate limits sufficient? | Negotiate higher limits |
| Storage | Growth projections? | Archival strategy |

### What Interviewers Are Looking For
- Methodical planning approach
- Understanding of scaling limitations
- Load testing awareness
- Risk mitigation thinking

### Tips for Answering
- Start with understanding current state
- Show specific scaling strategies per component
- Include timeline/phases
- Emphasize validation through testing

### Common Follow-Ups
- "How do you load test without affecting production?"
- "What if the database can't scale that much?"
- "How do you estimate costs for this scaling?"
- "What if the launch is moved up by 2 months?"

---

## 14. Feature Flag Incidents

### Question
**"What would you do if a feature flag was accidentally enabled in production, causing issues for all users?"**

### Sample Answer

**Immediate Response:**

**1. Disable the Flag:**
```python
# Most feature flag systems allow instant toggle
feature_flags.disable('problematic_feature')

# Or if using environment-based flags
# Update config and trigger redeploy/refresh
```

**2. Assess Impact:**
```
Questions:
- How long was the flag enabled?
- What functionality was affected?
- Were there any data changes?
- Which users were impacted?
```

**3. Verify Recovery:**
```
After disabling:
- Monitor error rates
- Check key metrics
- Verify affected functionality
- Confirm user-facing issues resolved
```

**Investigation:**

**4. Root Cause Analysis:**

**Common Causes:**

| Cause | Prevention |
|-------|------------|
| Manual error in dashboard | Access controls, approval workflow |
| Deployment included flag change | Separate flag from code deployment |
| Targeting rule mistake | Staging environment testing |
| Percentage rollout misconfigured | Default to 0%, require explicit enable |
| Stale flag state | Regular flag audits |

**5. Data Remediation (if needed):**

```python
# If the feature modified data incorrectly
def remediate_affected_data():
    # Find affected records
    affected_users = db.query("""
        SELECT DISTINCT user_id
        FROM audit_log
        WHERE feature_flag = 'problematic_feature'
        AND timestamp BETWEEN :start AND :end
    """)

    # Revert changes or apply corrections
    for user_id in affected_users:
        restore_previous_state(user_id)
```

**Prevention:**

**6. Feature Flag Best Practices:**

**Access Control:**
```yaml
# Role-based flag management
roles:
  developer:
    can_toggle: [development, staging]
    can_view: [production]

  release_manager:
    can_toggle: [all_environments]
    requires_approval: [production]
```

**Gradual Rollout:**
```python
# Safe rollout progression
rollout_stages = [
    {'percentage': 1, 'duration': '1h', 'monitor': True},
    {'percentage': 10, 'duration': '4h', 'monitor': True},
    {'percentage': 50, 'duration': '24h', 'monitor': True},
    {'percentage': 100, 'duration': 'ongoing', 'monitor': True}
]
```

**Circuit Breaker Integration:**
```python
@feature_flag('new_checkout')
@circuit_breaker(failure_threshold=5)
def process_checkout(cart):
    if feature_flags.is_enabled('new_checkout'):
        return new_checkout_flow(cart)
    else:
        return legacy_checkout_flow(cart)
```

**7. Operational Improvements:**

```
Implement:
- Audit logging for all flag changes
- Automatic rollback on error spike
- Staging environment that mirrors production flags
- Flag lifecycle management (cleanup old flags)
- Change approval workflow for production
```

### Feature Flag Safety Checklist

| Practice | Implementation |
|----------|---------------|
| Default to off | Flags start disabled in production |
| Audit logging | All changes logged with user/time |
| Approval workflow | Production changes need approval |
| Staged rollout | Percentage-based progressive enable |
| Automatic rollback | Error rate triggers disable |
| Regular cleanup | Remove old flags quarterly |
| Environment isolation | Separate configs per environment |

### What Interviewers Are Looking For
- Quick incident response
- Understanding of feature flag systems
- Prevention-focused improvements
- Knowledge of safe rollout practices

### Tips for Answering
- Lead with immediate fix (disable flag)
- Include impact assessment
- Show knowledge of feature flag best practices
- Discuss prevention measures

### Common Follow-Ups
- "How do you clean up old feature flags?"
- "How do you test feature flags before production?"
- "What metrics would trigger automatic flag disable?"
- "How do you handle feature flags in a microservices environment?"

---

## 15. Cache Failures

### Question
**"What would you do if your distributed cache (Redis cluster) suddenly became unavailable?"**

### Sample Answer

**Immediate Response:**

**1. Assess Impact:**
```
Check:
- Error rate increase
- Response time degradation
- Which services affected
- Is database handling the load?
```

**2. Enable Fallback:**
```python
# Graceful degradation pattern
def get_user_data(user_id):
    try:
        return cache.get(f"user:{user_id}")
    except CacheUnavailableError:
        # Fall back to database
        data = database.get_user(user_id)
        # Don't try to cache - cache is down
        return data
```

**3. Protect the Database:**
```python
# Implement rate limiting/circuit breaker for DB
@rate_limit(requests_per_second=1000)
@circuit_breaker(failure_threshold=10)
def get_user_data_with_protection(user_id):
    return database.get_user(user_id)
```

**Cache Recovery:**

**4. Diagnose Cache Issue:**

| Symptom | Possible Cause | Action |
|---------|---------------|--------|
| All nodes down | Infrastructure failure | Check cloud status, restart |
| Cluster split | Network partition | Check network, failover |
| Memory exhaustion | Data growth, leak | Eviction policy, increase memory |
| Connection exhaustion | Connection leak | Fix leaks, increase pool |
| Slow responses | Expensive operations | Optimize queries, add shards |

**5. Recovery Steps:**

**If infrastructure issue:**
```
1. Check cloud provider status
2. Verify network connectivity
3. Attempt node restart
4. If persistent, failover to backup cluster
```

**If data issue:**
```
1. Clear problematic keys
2. Adjust eviction policy
3. Increase memory
4. Enable cluster mode if not already
```

**6. Cache Warming:**
```python
# After cache recovery, warm critical data
def warm_cache():
    # Prioritize hot data
    hot_users = analytics.get_most_active_users(limit=10000)

    for user_id in hot_users:
        data = database.get_user(user_id)
        cache.set(f"user:{user_id}", data, ttl=3600)

    # Also warm common queries
    common_queries = get_frequently_accessed_data()
    for key, query in common_queries:
        result = database.execute(query)
        cache.set(key, result)
```

**Prevention:**

**7. Resilience Patterns:**

**Multi-layer Caching:**
```
Layer 1: Local in-memory cache (per instance)
Layer 2: Distributed cache (Redis cluster)
Layer 3: Database

If Layer 2 fails:
- Layer 1 absorbs some load
- Layer 3 handles remainder with protection
```

**Cache Replication:**
```
Primary Redis Cluster (Region A)
    |
    +--> Replica Cluster (Region B) [failover target]
```

**Request Coalescing:**
```python
# Prevent thundering herd on cache miss
class CoalescingCache:
    def __init__(self):
        self.in_flight = {}

    async def get_or_load(self, key, loader):
        if key in self.in_flight:
            return await self.in_flight[key]

        self.in_flight[key] = asyncio.create_task(loader(key))
        try:
            result = await self.in_flight[key]
            return result
        finally:
            del self.in_flight[key]
```

### Cache Failure Runbook

| Step | Action | Owner |
|------|--------|-------|
| 1 | Detect via monitoring alerts | Automation |
| 2 | Verify fallback is working | On-call |
| 3 | Enable database protection | On-call |
| 4 | Diagnose root cause | On-call |
| 5 | Recover or failover cache | On-call |
| 6 | Warm cache with critical data | On-call |
| 7 | Verify system stability | On-call |
| 8 | Post-mortem | Team |

### What Interviewers Are Looking For
- Understanding of cache dependencies
- Graceful degradation strategies
- Database protection awareness
- Knowledge of cache warming and thundering herd

### Tips for Answering
- Start with impact assessment
- Discuss protecting downstream systems (database)
- Mention cache warming for recovery
- Include prevention measures

### Common Follow-Ups
- "How do you prevent thundering herd on cache recovery?"
- "What's your cache invalidation strategy?"
- "How do you decide what to cache?"
- "How do you monitor cache effectiveness?"

---

## Summary

These scenario questions test your practical problem-solving abilities. Key principles across all scenarios:

1. **Assess before acting** - Understand the scope and impact first
2. **Prioritize user impact** - Contain and mitigate before root cause analysis
3. **Have multiple options** - No single solution fits all situations
4. **Communicate throughout** - Keep stakeholders informed
5. **Learn and prevent** - Every incident is an improvement opportunity

When answering scenario questions:
- Ask clarifying questions to show thoughtfulness
- Structure your response (assess, contain, fix, prevent)
- Discuss trade-offs between options
- Include monitoring and prevention
- Relate to your experience when possible
