# AWS Services Performance Troubleshooting Guide

## The Scenario: "Code is unchanged but service is slow"

**Possible Causes:**
1. AWS Infrastructure issues (EC2, RDS, ALB, etc.)
2. Misconfiguration (autoscaling, security groups, network)
3. Database issues (slow queries, capacity)
4. Load issues (spike in traffic)
5. Dependency issues (third-party APIs, network)
6. Resource exhaustion (CPU, memory, connections)

**Approach:** Systematic investigation starting from the outside-in

---

## **PHASE 1: ESTABLISH THE BASELINE (First 5 Minutes)**

### Step 1: Confirm the Slowness

```bash
# Check if it's really slow
time curl -w "@curl-format.txt" https://your-api.com/health

# Output:
#   Total time: 2500ms (baseline: 200ms) ← CONFIRM SLOWNESS
```

### Step 2: Check if it's Load or Latency

```bash
# If you're seeing high latency on all requests:
# LATENCY PROBLEM (infrastructure/app issue)

# If you're seeing SOME requests fast, SOME slow:
# LOAD/CAPACITY PROBLEM (too many requests)

# CloudWatch Insights query:
fields @timestamp, @duration, @message
| stats avg(@duration), max(@duration), pct(@duration, 95) by ispain
```

### Step 3: Check CloudWatch Dashboard

Go to CloudWatch and check:
- ✓ Application latency (APM dashboard)
- ✓ CPU usage (EC2 instances)
- ✓ Network in/out
- ✓ Error rate (check for errors causing slowness)
- ✓ Request count (is load spiking?)

---

## **PHASE 2: IDENTIFY THE CULPRIT (Next 10 Minutes)**

### Is It the Application Layer?

```bash
# SSH to EC2 instance
ssh -i key.pem ec2-user@instance-ip

# Check top processes
top -bn1 | head -20

# Expected: java process using normal CPU (20-40%)
# Problem: java using 90%+ → Application issue
# Problem: java barely using CPU but system slow → I/O or network issue
```

### Is It the Database?

```bash
# Check RDS performance via CLI
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=my-db \
  --start-time $(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Maximum

# Output:
# If connections maxed out (e.g., 100/100) → CONNECTION POOL EXHAUSTED
```

```bash
# Check query performance
# Go to RDS console → Performance Insights

# Things to look for:
# - Long-running queries (>5 seconds)
# - High lock waits
# - I/O spikes
# - CPU percentage of database
```

### Is It the Load Balancer?

```bash
# Check ALB response time
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name TargetResponseTime \
  --dimensions Name=LoadBalancer,Value=app/my-alb/1234567890123456 \
  --start-time $(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Average,Maximum
```

### Is It the Network?

```bash
# Check security group rules (maybe ingress/egress is misconfigured)
aws ec2 describe-security-groups --group-ids sg-xxxxxx

# Check network performance on instance
ec2-user$ sar -n DEV 1 5  # Check network I/O

# Look for:
# rxpck/s → incoming packets (should be reasonable)
# txpck/s → outgoing packets (should be reasonable)
# If zero or extremely high → network issue
```

---

## **PHASE 3: DEEP DIVE BY SERVICE**

## **If Problem is EC2 Instance Slow**

### Check 1: CPU Usage

```bash
# CLI Method
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average,Maximum
```

**If CPU is high (>80%):**
- Application doing CPU-intensive work?
- Unoptimized queries?
- Infinite loop in code?
  
```bash
# On the instance, see what's consuming CPU
top -o %CPU
# or
ps aux --sort=-%cpu | head
```

**If CPU is low (<20%) but service is slow:**
- I/O bound (database, disk, network)
- Waiting on external service
- Not enough parallelism (threads)

### Check 2: Memory Usage

```bash
# CLI for memory metrics
aws cloudwatch get-metric-statistics \
  --namespace CWAgent \
  --metric-name mem_percent_used \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average
```

**If memory is high (>85%):**
- Memory leak in application
- Cache growing too large
- Need to restart or increase instance size

```bash
# On the instance, check memory
free -h
# or
ps aux --sort=-%mem | head
```

### Check 3: Disk I/O

```bash
# On the instance
iostat -x 1 5  # Check disk utilization

# Look for:
# %util > 90% → disk is bottleneck
# await (average wait time) → if >50ms, disk is slow
```

### Check 4: Thread Count

```bash
# For Java application
jps -l  # List Java processes
jstack <pid> | grep "tid" | wc -l  # Count threads

# Too many threads (>1000):
# - Thread leak
# - Thread pool misconfigured
# - Each request creating new threads
```

### Check 5: Garbage Collection (GC) Pauses

```bash
# Enable GC logging (if not already)
# Add to Java startup flags:
# -Xlog:gc*:file=gc.log:time,level,tags

# Analyze GC logs
tail -f gc.log

# Look for:
# Full GC happening frequently (should be rare)
# Pause time > 1 second (should be <100ms)
# Heap nearly full (90%+)
```

Java/Spring Boot code to check GC:

```java
// Check GC stats at runtime
import java.lang.management.GarbageCollectorMXBean;
import java.lang.management.ManagementFactory;

public class GCMonitor {
    public static void printGCStats() {
        for (GarbageCollectorMXBean gc : ManagementFactory.getGarbageCollectorMXBeans()) {
            System.out.println("GC: " + gc.getName());
            System.out.println("  Collection count: " + gc.getCollectionCount());
            System.out.println("  Collection time: " + gc.getCollectionTime() + "ms");
            
            if (gc.getCollectionCount() > 100) {
                System.out.println("  ⚠️ WARNING: Frequent GC!");
            }
        }
    }
}
```

---

## **If Problem is RDS (Database) Slow**

### Check 1: Connection Pool Exhaustion

```java
// Spring Boot: Check HikariCP pool stats
@RestController
public class HealthController {
    @Autowired
    private HikariDataSource dataSource;
    
    @GetMapping("/pool-stats")
    public Map<String, Object> getPoolStats() {
        return Map.of(
            "active_connections", dataSource.getHikariPoolMXBean().getActiveConnections(),
            "idle_connections", dataSource.getHikariPoolMXBean().getIdleConnections(),
            "total_connections", dataSource.getHikariPoolMXBean().getTotalConnections(),
            "pending_threads", dataSource.getHikariPoolMXBean().getThreadsAwaitingConnection()
        );
    }
    
    // Example output:
    // {
    //   "active_connections": 18,
    //   "idle_connections": 2,
    //   "total_connections": 20,
    //   "pending_threads": 150  ← PROBLEM! Threads waiting for connection
    // }
}
```

**If pending_threads > 0:**
- Pool exhausted
- Queries are slow (connections not being returned)
- Need to:
  1. Increase pool size
  2. Optimize slow queries
  3. Find connection leaks

### Check 2: Slow Queries

```sql
-- On RDS, enable slow query log
-- Then query the log

-- CloudWatch Insights for slow queries:
fields @timestamp, @duration, @message
| filter @duration > 1000
| stats count() as slow_queries
```

### Check 3: RDS Performance Insights

```bash
# Via CLI
aws pi get-resource-metrics \
  --service-type RDS \
  --identifier-arn arn:aws:rds:region:account:db:dbname \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period-in-seconds 60 \
  --metric-queries '[{"Metric":"db.load"}]'
```

**Or via Console:**
```
RDS → Databases → Select DB → Performance Insights
Look for:
- High db.load (>1 = saturated)
- Top SQL queries
- Wait events
```

### Check 4: RDS CPU and Storage

```bash
# Check RDS CPU
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=my-db \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average,Maximum

# Check IOPS (disk reads/writes)
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name ReadIOPS \
  --dimensions Name=DBInstanceIdentifier,Value=my-db \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average,Maximum
```

**If IOPS are maxed out:**
```
Current: db.t3.medium (burst capable)
→ Upgrade to db.r5.xlarge (higher IOPS)
→ Or enable Provisioned IOPS
```

---

## **If Problem is Load Balancer (ALB)**

```bash
# Check ALB health
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:region:account:targetgroup/name/id

# Look for:
# - UnhealthyThreshold: EC2 instances marked unhealthy?
# - HealthyHostCount vs UnhealthyHostCount
```

```bash
# Check ALB latency
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name TargetResponseTime \
  --dimensions Name=LoadBalancer,Value=app/my-alb/1234567890123456 \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Average,Maximum
```

---

## **PHASE 4: ROOT CAUSE SCENARIOS & FIXES**

### Scenario 1: Database Connection Pool Exhausted

**Symptoms:**
- Response time 500ms→5000ms
- Pending threads in connection pool
- Database CPU low, but queries waiting

**Fix:**

```java
// application.yml
spring:
  datasource:
    hikari:
      maximum-pool-size: 30  # Increase from default 10
      minimum-idle: 5        # Keep idle connections ready
      connection-timeout: 30000  # 30 second timeout
      idle-timeout: 600000       # 10 minutes
      max-lifetime: 1800000      # 30 minutes
      
      # New in newer Spring Boot - connection validation
      connection-test-query: "SELECT 1"
```

Or via Spring Boot 3.0+ properties:
```yaml
spring:
  datasource:
    hikari:
      pool-size: 30
      validation-query: "SELECT 1"
```

### Scenario 2: Slow Database Queries

**Symptoms:**
- Database CPU high (80%+)
- Individual queries taking >1 second
- No code changes

**Investigation:**

```sql
-- Find top 10 slow queries
SELECT 
  query,
  count(*) as times_executed,
  avg(duration) as avg_duration_ms,
  max(duration) as max_duration_ms
FROM performance_schema.events_statements_summary_by_digest
ORDER BY avg_duration_ms DESC
LIMIT 10;
```

**Fix:**

```sql
-- Add indexes for frequently slow queries
EXPLAIN SELECT * FROM orders WHERE user_id = 123 AND status = 'pending';
-- If "Full table scan" → Add index

CREATE INDEX idx_orders_user_status 
ON orders(user_id, status);
```

### Scenario 3: Autoscaling Not Kicking In

**Symptoms:**
- CPU or load balancer is at 100%
- But no new instances launching
- Response time degrading

**Check:**

```bash
# Check autoscaling group
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names my-asg

# Look for:
# - DesiredCapacity vs CurrentSize (mismatch = problem)
# - Instances being launched but in "Pending" state
# - Scaling policies misconfigured
```

**Fix:**

```bash
# Check scaling policies
aws autoscaling describe-policies \
  --auto-scaling-group-name my-asg

# Example: Scaling policy uses 80% CPU threshold
# If threshold too high, add more aggressive policy:
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name my-asg \
  --policy-name scale-up-aggressive \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
      "TargetValue": 50.0,
      "PredefinedMetricSpecification": {
        "PredefinedMetricType": "ASGAverageCPUUtilization"
      }
    }'
```

### Scenario 4: Memory Leak in Application

**Symptoms:**
- Consistent memory growth over hours
- Eventually OutOfMemoryError
- Service degradation before crash

**Investigation:**

```bash
# SSH to instance
# Check memory trend
watch -n 5 'free -h'

# If seeing consistent growth:
# Generate heap dump
jmap -dump:live,format=b,file=heap.bin <pid>

# Download and analyze with Eclipse MAT
# Or use command line:
jhat -J-Xmx3g heap.bin
```

```java
// Or use Spring Boot Actuator
// GET http://localhost:8080/actuator/heapdump
// Downloads heap dump automatically
```

**Fix:**
- Find object allocations not being freed
- Add explicit cleanup
- Or increase heap size as temporary fix

```bash
# Increase JVM heap
# In application startup:
export JAVA_OPTS="-Xms2g -Xmx4g"
java $JAVA_OPTS -jar app.jar
```

### Scenario 5: Network/Security Group Misconfiguration

**Symptoms:**
- Connection timeouts
- Intermittent slowness
- But no CPU/memory issues

**Check:**

```bash
# From EC2 instance, test connectivity
nc -zv rds-endpoint.amazonaws.com 3306  # Check RDS
nc -zv external-api.com 443             # Check external API

# If timeout → Security group blocks it

# Check security group rules
aws ec2 describe-security-groups --group-ids sg-xxxxxx

# Should allow:
# - Inbound: Port 8080 from ALB (or 0.0.0.0/0)
# - Outbound: 443 to external APIs
```

**Fix:**

```bash
# Add missing outbound rule
aws ec2 authorize-security-group-egress \
  --group-id sg-xxxxxx \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0
```

### Scenario 6: Lambda Cold Starts

**Symptoms (if using Lambda):**
- First request slow (5-10 seconds)
- Subsequent requests fast
- Intermittent slowness

**Fix:**

```java
// Java Lambda: Pre-compile at build time
// pom.xml or build.gradle
// Use GraalVM native-image to reduce cold start from 5s to 100ms

// Or use Provisioned Concurrency
aws lambda put-provisioned-concurrency-config \
  --function-name my-function \
  --provisioned-concurrent-executions 10
```

---

## **PHASE 5: MONITORING SETUP (Prevent Future Issues)**

### Set Up Comprehensive Monitoring

```bash
# CloudWatch custom metric for application latency
aws cloudwatch put-metric-data \
  --metric-name ApplicationLatency \
  --namespace MyApp \
  --value 450 \
  --unit Milliseconds \
  --dimensions Environment=Production,Service=API
```

```java
// Spring Boot: Send metrics to CloudWatch
@Component
public class LatencyMonitor {
    @Autowired
    private MeterRegistry meterRegistry;
    
    @PostMapping("/api/order")
    public ResponseEntity createOrder(@RequestBody Order order) {
        long startTime = System.currentTimeMillis();
        
        try {
            // Process order
            return ResponseEntity.ok(processOrder(order));
        } finally {
            long duration = System.currentTimeMillis() - startTime;
            meterRegistry.timer("api.order.creation").record(duration, TimeUnit.MILLISECONDS);
        }
    }
}
```

### Set Up Alarms

```bash
# Alert if response time > 1 second
aws cloudwatch put-metric-alarm \
  --alarm-name HighLatencyAlert \
  --alarm-description "Alert if latency > 1000ms" \
  --metric-name TargetResponseTime \
  --namespace AWS/ApplicationELB \
  --statistic Average \
  --period 300 \
  --threshold 1000 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:region:account:topic-name
```

### Set Up X-Ray Tracing

```java
// Spring Boot with X-Ray
// Add dependency: spring-cloud-starter-aws-xray

@Configuration
public class XRayConfig {
    @Bean
    public AWSXRay xray() {
        return AWSXRayRecorderBuilder.standard().build();
    }
}

// Now all requests automatically traced
// See latency breakdown:
// API → Database → Cache → External Service
```

---

## **TROUBLESHOOTING CHECKLIST**

### Minute 1-5: Confirm & Gather Data
- [ ] Confirm actual slowness (not just perception)
- [ ] Check CloudWatch dashboard
- [ ] Note error rates
- [ ] Check traffic spike (is load actually higher?)

### Minute 5-15: Identify Layer
- [ ] Application layer: CPU/Memory/Threads/GC
- [ ] Database layer: Connections/Queries/IOPS
- [ ] Network layer: Security groups/VPC/Internet
- [ ] Load balancer: Health check/Configuration

### Minute 15-30: Deep Dive
- [ ] Analyze slow queries
- [ ] Check connection pool
- [ ] Review recent changes (infrastructure)
- [ ] Check CloudTrail for configuration changes

### Minute 30+: Implement Fix
- [ ] Increase pool size / instance size
- [ ] Add indexes to database
- [ ] Optimize slow queries
- [ ] Scale up / adjust autoscaling
- [ ] Fix security group rules

---

## **FASTEST WINS (Quick Fixes)**

| Problem | Symptom | Quick Fix | Time |
|---------|---------|-----------|------|
| Connection pool exhausted | Pending threads | Increase HikariCP `maximum-pool-size` | 2 min |
| Slow query | DB CPU high, one query slow | Add index | 5 min |
| Autoscaling not working | CPU 100%, no scaling | Lower CPU threshold | 3 min |
| Memory leak | Memory growing | Restart instance | 2 min |
| Security group block | Timeout to external API | Add egress rule | 2 min |
| Instance undersized | CPU consistently 90%+ | Upgrade instance type | 10 min |
| Misconfigured ALB | Requests hanging | Fix health check | 5 min |

---

## **TOOLS CHEAT SHEET**

```bash
# CloudWatch Logs Insights
# Find slow requests
fields @timestamp, @duration, @message
| filter @duration > 5000
| stats count() by @message

# Find errors
fields @timestamp, @message
| filter @message like /ERROR/
| stats count() by @message

# Database connections
SELECT current_connections FROM sys.sysprocesses
SELECT COUNT(*) FROM information_schema.processlist

# Java process diagnostics
jps -l                    # List Java processes
jstack <pid>              # Thread dump
jmap -heap <pid>          # Heap info
jstat -gc -h10 <pid> 1000 # GC stats every 1 second
jcmd <pid> GC.class_stats  # Class statistics
```

---

## **REMEMBER**

1. **Systematic approach beats guessing**
2. **Metrics don't lie** - trust CloudWatch over intuition
3. **Check external dependencies** - third-party APIs often culprit
4. **Always compare to baseline** - "slow" is relative
5. **Enable monitoring BEFORE problems** - retroactive analysis is hard
6. **Document what you find** - helps next time

**Most Common Causes (in order):**
1. Database connection pool exhausted (40%)
2. Slow database queries (30%)
3. Insufficient instance size (15%)
4. Memory leak (10%)
5. Network/security group issue (5%)
