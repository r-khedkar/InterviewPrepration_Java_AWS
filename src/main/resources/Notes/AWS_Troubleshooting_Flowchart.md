# AWS Performance Issues - Quick Diagnosis Flowchart

## START HERE: Is Service Actually Slow?

```
User reports "service is slow"
        ↓
   Run: time curl https://your-api.com/health
        ↓
   Is response time > 2x normal?
   ├─ NO → false alarm, check baseline metrics
   └─ YES → Continue to next step
```

---

## DECISION TREE: WHERE IS THE PROBLEM?

```
Service is slow (confirmed)
        ↓
   Check CloudWatch Dashboard (all metrics)
        ↓
   ┌──────────────────────────────────────────────────┐
   │ What's elevated?                                 │
   └──────────────────────────────────────────────────┘
        ↓
   ┌────────────────────────────────────────────────┐
   │ A) High CPU, High Memory, High Thread Count   │
   ├────────────────────────────────────────────────┤
   │ B) High Database Connections, DB CPU High     │
   ├────────────────────────────────────────────────┤
   │ C) Normal CPU/Memory but still slow            │
   ├────────────────────────────────────────────────┤
   │ D) Everything normal but requests timing out  │
   ├────────────────────────────────────────────────┤
   │ E) Intermittent slowness, sometimes fast      │
   └────────────────────────────────────────────────┘
```

---

## PATH A: APPLICATION LAYER ISSUE

```
High CPU / High Memory / High Thread Count
        ↓
   ┌─────────────────────────────────────────────┐
   │ Check: top -bn1 | head -20                 │
   └─────────────────────────────────────────────┘
        ↓
   Is CPU usage 80%+ on single process?
   ├─ YES (CPU bound)
   │  ├─ Check: jstack <pid> | grep "runnable"
   │  ├─ Look for: Infinite loops, tight loops
   │  ├─ Solution: Optimize algorithm or scale horizontally
   │  └─ Time to fix: 15-60 minutes
   │
   └─ NO (I/O bound or thread issue)
      ├─ Check: jstack <pid> | grep -c "tid"
      ├─ If thread count > 2000
      │  ├─ Problem: Thread leak or misconfigured thread pool
      │  ├─ Solution: Increase pool size, fix leak
      │  └─ Time to fix: 10 minutes
      │
      └─ If memory growing
         ├─ Problem: Memory leak
         ├─ Solution: Heap dump analysis, restart, increase heap
         └─ Time to fix: 20 minutes
```

**Quick Fixes for Path A:**
```bash
# Restart (temporary fix)
docker restart <container>

# Increase JVM memory
export JAVA_OPTS="-Xms2g -Xmx4g"
java $JAVA_OPTS -jar app.jar

# Check for infinite loops
jstack <pid> > thread-dump.txt
# Look for threads in same method repeatedly
```

---

## PATH B: DATABASE ISSUE

```
High Database Connections / High DB CPU
        ↓
   ┌─────────────────────────────────────────────┐
   │ Check: 
   │ - Connection pool utilization
   │ - Database CPU %
   │ - Top queries (Performance Insights)
   └─────────────────────────────────────────────┘
        ↓
   Is connection pool exhausted (pending threads > 0)?
   ├─ YES
   │  ├─ Problem: Queries slow, connections not returned quickly
   │  ├─ Solution: Increase HikariCP pool size
   │  ├─ application.yml:
   │  │  spring:
   │  │    datasource:
   │  │      hikari:
   │  │        maximum-pool-size: 30  # Increase from 10
   │  │        connection-timeout: 30000
   │  └─ Time to fix: 5 minutes (restart app)
   │
   └─ NO
      ├─ Check: Individual query performance (Performance Insights)
      │
      ├─ Is one query taking >1000ms?
      │  ├─ YES
      │  │  ├─ Problem: Slow query
      │  │  ├─ Run: EXPLAIN on that query
      │  │  ├─ Solution: Add index or optimize query
      │  │  └─ Time to fix: 15-30 minutes
      │  │
      │  └─ NO (all queries fast but DB CPU high)
      │     ├─ Problem: High query volume OR DB instance too small
      │     ├─ Check: ReadIOPS and WriteIOPS
      │     ├─ Solution: Increase IOPS or scale DB
      │     └─ Time to fix: 10-30 minutes
```

**Quick Fixes for Path B:**
```bash
# Emergency: Increase pool size immediately
# In application.yml (restart app)
spring:
  datasource:
    hikari:
      maximum-pool-size: 50  # Increase

# Check slow query log
# RDS Console → Logs → slow-query log

# Quick: Restart database (not recommended but works)
aws rds reboot-db-instance --db-instance-identifier my-db

# Better: Upgrade instance type
aws rds modify-db-instance \
  --db-instance-identifier my-db \
  --db-instance-class db.r5.2xlarge \
  --apply-immediately
```

---

## PATH C: I/O BOUND (Slow but CPU low)

```
CPU/Memory normal, but service still slow
        ↓
   ┌─────────────────────────────────────────────┐
   │ Check:
   │ - Is app waiting on database? (profiler)
   │ - Is app waiting on external API? (logs)
   │ - Is disk I/O high? (iostat)
   │ - Is network latency high? (cloudwatch)
   └─────────────────────────────────────────────┘
        ↓
   Likely Culprits:
   ├─ Slow external API call
   │  └─ Solution: Add caching or timeout
   │
   ├─ Database query latency high
   │  └─ Solution: Optimize query or use Redis cache
   │
   ├─ Disk I/O bottleneck
   │  └─ Solution: Use EBS optimized instance, more IOPS
   │
   └─ Network latency (cross-region)
      └─ Solution: Use closer region or VPC endpoint
```

**Quick Fixes for Path C:**
```bash
# Add timeout to external API calls (Java)
@Timeout(millis = 5000)  // Max 5 seconds
public String callExternalAPI() {
    // Add timeout
}

# Add Redis cache
@Cacheable("product_cache")
public Product getProduct(Long id) {
    return db.findById(id);
}

# Check network latency
ping -c 5 external-api.com
```

---

## PATH D: EVERYTHING NORMAL BUT TIMEOUTS

```
CPU/Memory/DB all normal but requests timeout
        ↓
   ┌─────────────────────────────────────────────┐
   │ Check:
   │ - Security group rules
   │ - Network ACLs
   │ - Route tables
   │ - NACLs on subnets
   └─────────────────────────────────────────────┘
        ↓
   From EC2, test connectivity:
   ├─ nc -zv rds-endpoint.amazonaws.com 3306
   ├─ nc -zv external-api.com 443
   ├─ curl http://alb-endpoint.com:8080/health
   │
   ├─ Connection refused? → Port not open
   │  └─ Solution: Add security group ingress/egress rule
   │
   ├─ Connection timeout? → Firewall/NACL blocking
   │  └─ Solution: Check NACLs, adjust ephemeral ports
   │
   └─ Connection OK but slow? → See Path C
```

**Quick Fixes for Path D:**
```bash
# Add outbound rule to security group
aws ec2 authorize-security-group-egress \
  --group-id sg-xxxxxx \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# Add inbound rule for ALB
aws ec2 authorize-security-group-ingress \
  --group-id sg-app \
  --protocol tcp \
  --port 8080 \
  --source-security-group-id sg-alb
```

---

## PATH E: INTERMITTENT SLOWNESS (Bursty)

```
Sometimes fast, sometimes slow, random pattern
        ↓
   ┌─────────────────────────────────────────────┐
   │ Likely causes:
   │ - Garbage collection pause (Java)
   │ - Autoscaling kicking in
   │ - Cache miss spike
   │ - Cold start (Lambda)
   │ - Database connection reset
   └─────────────────────────────────────────────┘
        ↓
   Check: Does slow request coincide with GC?
   ├─ YES
   │  ├─ Solution: Increase heap size, tune GC algorithm
   │  └─ Check: -Xms equal to -Xmx (prevents resize pause)
   │
   ├─ Check: Does slow request coincide with scale-up?
   │  ├─ YES
   │  │  ├─ Solution: Lower autoscaling threshold
   │  │  └─ or Increase initial capacity
   │  │
   │  └─ Check: Cache hit ratio
   │     ├─ If low, implement caching strategy
   │     └─ Solution: Warm up cache on startup
```

**Quick Fixes for Path E:**
```bash
# Check if GC is happening
tail -f gc.log | grep "GC pause"

# Fix for Java GC pauses
# Add to startup flags:
-XX:+UseG1GC           # Use G1GC
-XX:MaxGCPauseMillis=200  # Max 200ms pause

# Lower autoscaling threshold
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name my-asg \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
      "TargetValue": 50.0,  # Lower from 80%
      "PredefinedMetricSpecification": {
        "PredefinedMetricType": "ASGAverageCPUUtilization"
      }
    }'
```

---

## METRIC QUICK REFERENCE

| Metric | Good Range | Bad Range | Action |
|--------|-----------|-----------|--------|
| CPU Utilization | 20-60% | >80% | Scale up or optimize |
| Memory | 40-70% used | >85% | Increase heap or find leak |
| DB Connections | 30-50% of max | >80% | Increase pool size |
| Response Time | <200ms | >1000ms | Investigate bottleneck |
| GC Pause Time | <100ms | >500ms | Tune GC or increase heap |
| Thread Count | <500 | >1500 | Find thread leak |
| Request Queue | 0 | >100 | Add capacity |
| Cache Hit Ratio | >80% | <60% | Improve caching |

---

## TOOLS TO RUN FIRST (In Order)

### 1. Check Metrics (1 minute)
```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-xxx \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average,Maximum
```

### 2. Check Application Logs (2 minutes)
```bash
aws logs tail /aws/lambda/my-function --follow --since 15m

# Or for container logs
docker logs -f <container_id> --tail 50
```

### 3. Check Database (2 minutes)
```bash
# RDS Console → Performance Insights
# Look for: High db.load, slow queries
```

### 4. Check Infrastructure (3 minutes)
```bash
# SSH to instance
top -bn1 | head -20      # CPU/Memory
ps aux | grep java        # Application process
free -h                   # Memory breakdown
df -h                     # Disk usage
```

### 5. Check Application Profiler (5 minutes)
```bash
# Java
jstack <pid> > thread-dump.txt
# Search for: "runnable", "waiting on", "locked"

# Or use built-in endpoint
curl http://localhost:8080/actuator/health/liveness
curl http://localhost:8080/actuator/metrics
```

---

## COMMON ISSUES DECISION TABLE

| Symptom | Most Likely | Second Likely | Check First |
|---------|-------------|---------------|-------------|
| Slowly getting worse over time | Memory leak | Cache growing | `jstat -gc <pid>` |
| Sudden spike in latency | Database query | Autoscaling | `aws cloudwatch` |
| Intermittent timeouts | Network/Security | Database connection | `nc -zv` |
| High error rate + slow | Application crash | Database down | Check logs |
| Responses getting slower | Connection pool | Disk I/O | Check HikariCP |
| Only certain endpoints slow | Query N+1 problem | Missing index | Query execution plan |

---

## GOLDEN RULES

1. **Metrics First** - Don't guess, look at data
2. **Baseline Matters** - Know normal before diagnosing abnormal
3. **Isolate Variables** - Change one thing at a time
4. **Monitor External** - Your code can't slow down external APIs
5. **Cache Everything** - 90% of slowness is cache misses
6. **Connection Pools** - 40% of production issues are pool related
7. **Document** - Write it down for next time
8. **Automate** - If you diagnose it twice, automate the fix

---

## ESCALATION CHECKLIST

If slow after 30 minutes:
- [ ] Notified team lead
- [ ] Enabled verbose logging
- [ ] Captured heap dump
- [ ] Captured thread dump
- [ ] Saved performance metrics
- [ ] Noted exact time of issue

If slow after 1 hour:
- [ ] Page on-call engineer
- [ ] Consider emergency scaling
- [ ] Start failover if critical
- [ ] Engage AWS support (premium)

---

**REMEMBER: The fastest diagnosis comes from good monitoring. Set up alarms BEFORE you need them.**
