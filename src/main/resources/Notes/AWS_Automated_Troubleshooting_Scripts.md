# AWS Performance Troubleshooting - Automated Scripts

## Script 1: Quick Diagnostics Bash Script

Save as: `diagnose.sh`

```bash
#!/bin/bash

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

echo -e "${BLUE}=== AWS Performance Diagnostics ===${NC}\n"

# Configuration
INSTANCE_ID=${1:-"i-1234567890abcdef0"}
BUCKET_NAME="my-bucket"
DB_INSTANCE="my-db"
REGION="us-east-1"

# Function to print section
print_section() {
    echo -e "\n${BLUE}>>> $1${NC}"
}

# Function to print result
print_result() {
    local metric=$1
    local value=$2
    local threshold=$3
    
    if [ -z "$threshold" ]; then
        echo -e "${GREEN}✓${NC} $metric: $value"
    elif (( $(echo "$value > $threshold" | bc -l) )); then
        echo -e "${RED}✗${NC} $metric: $value (Threshold: $threshold)"
    else
        echo -e "${GREEN}✓${NC} $metric: $value"
    fi
}

# ===== CHECK 1: EC2 Instance CPU =====
print_section "1. EC2 CPU Utilization (Last 5 minutes)"

CPU=$(aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=$INSTANCE_ID \
  --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average \
  --region $REGION \
  --query 'Datapoints[0].Average' \
  --output text)

print_result "CPU Usage" "${CPU}%" "80"

# ===== CHECK 2: RDS Database Connections =====
print_section "2. RDS Database Connections"

DB_CONNECTIONS=$(aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=$DB_INSTANCE \
  --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Maximum \
  --region $REGION \
  --query 'Datapoints[0].Maximum' \
  --output text)

print_result "DB Connections" "$DB_CONNECTIONS" "80"

# ===== CHECK 3: RDS CPU =====
print_section "3. RDS CPU Utilization"

RDS_CPU=$(aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=$DB_INSTANCE \
  --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average \
  --region $REGION \
  --query 'Datapoints[0].Average' \
  --output text)

print_result "DB CPU Usage" "${RDS_CPU}%" "80"

# ===== CHECK 4: ALB Response Time =====
print_section "4. Application Load Balancer Response Time"

ALB_RESPONSE=$(aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name TargetResponseTime \
  --dimensions Name=LoadBalancer,Value="app/my-alb/1234567890123456" \
  --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average \
  --region $REGION \
  --query 'Datapoints[0].Average' \
  --output text)

print_result "ALB Response Time" "${ALB_RESPONSE}s" "1"

# ===== CHECK 5: Target Health =====
print_section "5. ALB Target Health"

HEALTHY=$(aws elbv2 describe-target-health \
  --target-group-arn "arn:aws:elasticloadbalancing:$REGION:account:targetgroup/my-tg/abc123" \
  --query 'length(TargetHealthDescriptions[?TargetHealth.State==`healthy`])' \
  --output text)

UNHEALTHY=$(aws elbv2 describe-target-health \
  --target-group-arn "arn:aws:elasticloadbalancing:$REGION:account:targetgroup/my-tg/abc123" \
  --query 'length(TargetHealthDescriptions[?TargetHealth.State==`unhealthy`])' \
  --output text)

echo -e "${GREEN}✓${NC} Healthy targets: $HEALTHY"
if [ "$UNHEALTHY" -gt 0 ]; then
    echo -e "${RED}✗${NC} Unhealthy targets: $UNHEALTHY"
fi

# ===== CHECK 6: RDS IOPS =====
print_section "6. RDS I/O Operations"

RDS_IOPS=$(aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name ReadIOPS \
  --dimensions Name=DBInstanceIdentifier,Value=$DB_INSTANCE \
  --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average \
  --region $REGION \
  --query 'Datapoints[0].Average' \
  --output text)

echo -e "${GREEN}✓${NC} RDS Read IOPS: $RDS_IOPS"

# ===== CHECK 7: Autoscaling Status =====
print_section "7. Autoscaling Group Status"

ASG_NAME="my-asg"
DESIRED=$(aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names $ASG_NAME \
  --region $REGION \
  --query 'AutoScalingGroups[0].DesiredCapacity' \
  --output text)

CURRENT=$(aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names $ASG_NAME \
  --region $REGION \
  --query 'length(AutoScalingGroups[0].Instances)' \
  --output text)

if [ "$DESIRED" -eq "$CURRENT" ]; then
    echo -e "${GREEN}✓${NC} Desired: $DESIRED, Current: $CURRENT (in sync)"
else
    echo -e "${YELLOW}⚠${NC} Desired: $DESIRED, Current: $CURRENT (scaling in progress)"
fi

# ===== CHECK 8: Recent Errors =====
print_section "8. Recent Application Errors (Last 15 minutes)"

ERROR_COUNT=$(aws logs tail /aws/lambda/my-function \
  --since 15m \
  --filter-pattern "ERROR" \
  --format short 2>/dev/null | wc -l)

if [ "$ERROR_COUNT" -gt 0 ]; then
    echo -e "${RED}✗${NC} Found $ERROR_COUNT errors in logs"
else
    echo -e "${GREEN}✓${NC} No errors detected"
fi

# ===== SUMMARY =====
print_section "Summary"
echo "Run the following for detailed diagnostics:"
echo "1. Check CloudWatch dashboard"
echo "2. Check RDS Performance Insights"
echo "3. Check CloudWatch Logs Insights"
echo "4. Review recent changes in CloudTrail"
echo "5. SSH to instance and run: top, jstack, jmap"

```

**Usage:**
```bash
chmod +x diagnose.sh
./diagnose.sh i-1234567890abcdef0
```

---

## Script 2: Python Comprehensive Diagnostic Script

Save as: `diagnose.py`

```python
#!/usr/bin/env python3

import boto3
import json
from datetime import datetime, timedelta
from botocore.exceptions import ClientError

class AWSPerformanceDiagnostics:
    def __init__(self, region='us-east-1'):
        self.region = region
        self.cw_client = boto3.client('cloudwatch', region_name=region)
        self.rds_client = boto3.client('rds', region_name=region)
        self.ec2_client = boto3.client('ec2', region_name=region)
        self.asg_client = boto3.client('autoscaling', region_name=region)
        self.elbv2_client = boto3.client('elbv2', region_name=region)
        self.logs_client = boto3.client('logs', region_name=region)
        
        self.issues_found = []
    
    def get_metric(self, namespace, metric_name, dimensions, stat='Average'):
        """Get latest metric value"""
        try:
            response = self.cw_client.get_metric_statistics(
                Namespace=namespace,
                MetricName=metric_name,
                Dimensions=dimensions,
                StartTime=datetime.utcnow() - timedelta(minutes=15),
                EndTime=datetime.utcnow(),
                Period=300,
                Statistics=[stat]
            )
            
            if response['Datapoints']:
                datapoint = sorted(response['Datapoints'], 
                                 key=lambda x: x['Timestamp'])[-1]
                return float(datapoint[stat])
            return None
        except ClientError as e:
            print(f"❌ Error getting {metric_name}: {e}")
            return None
    
    def check_ec2_cpu(self, instance_id):
        """Check EC2 CPU utilization"""
        print("\n[1] EC2 CPU Utilization")
        print("-" * 50)
        
        cpu = self.get_metric(
            namespace='AWS/EC2',
            metric_name='CPUUtilization',
            dimensions=[{'Name': 'InstanceId', 'Value': instance_id}]
        )
        
        if cpu is None:
            print("⚠️  Could not retrieve CPU metric")
            return
        
        print(f"CPU Usage: {cpu:.2f}%")
        
        if cpu > 80:
            print("🔴 HIGH: CPU utilization critical")
            self.issues_found.append(f"EC2 CPU high: {cpu:.2f}%")
        elif cpu > 60:
            print("🟡 MODERATE: CPU utilization elevated")
        else:
            print("✅ NORMAL: CPU utilization healthy")
    
    def check_rds_connections(self, db_instance):
        """Check RDS connection pool"""
        print("\n[2] RDS Database Connections")
        print("-" * 50)
        
        connections = self.get_metric(
            namespace='AWS/RDS',
            metric_name='DatabaseConnections',
            dimensions=[{'Name': 'DBInstanceIdentifier', 'Value': db_instance}],
            stat='Maximum'
        )
        
        if connections is None:
            print("⚠️  Could not retrieve connection metric")
            return
        
        print(f"Active Connections: {int(connections)}")
        
        # Get max connections for the DB
        try:
            db_info = self.rds_client.describe_db_instances(
                DBInstanceIdentifier=db_instance
            )
            # This is approximate
            max_connections = 100  # varies by instance type
            
            utilization = (connections / max_connections) * 100
            print(f"Connection Pool Utilization: {utilization:.1f}%")
            
            if utilization > 80:
                print("🔴 HIGH: Connection pool nearly exhausted")
                self.issues_found.append(f"DB connections high: {utilization:.1f}%")
            elif utilization > 60:
                print("🟡 MODERATE: Connection pool usage high")
            else:
                print("✅ NORMAL: Connection pool healthy")
                
        except ClientError as e:
            print(f"Could not get DB instance info: {e}")
    
    def check_rds_cpu(self, db_instance):
        """Check RDS CPU utilization"""
        print("\n[3] RDS CPU Utilization")
        print("-" * 50)
        
        cpu = self.get_metric(
            namespace='AWS/RDS',
            metric_name='CPUUtilization',
            dimensions=[{'Name': 'DBInstanceIdentifier', 'Value': db_instance}]
        )
        
        if cpu is None:
            print("⚠️  Could not retrieve RDS CPU metric")
            return
        
        print(f"Database CPU: {cpu:.2f}%")
        
        if cpu > 80:
            print("🔴 HIGH: Database CPU critical")
            self.issues_found.append(f"RDS CPU high: {cpu:.2f}%")
        elif cpu > 60:
            print("🟡 MODERATE: Database CPU elevated (check for slow queries)")
        else:
            print("✅ NORMAL: Database CPU healthy")
    
    def check_rds_iops(self, db_instance):
        """Check RDS I/O operations"""
        print("\n[4] RDS I/O Performance")
        print("-" * 50)
        
        read_iops = self.get_metric(
            namespace='AWS/RDS',
            metric_name='ReadIOPS',
            dimensions=[{'Name': 'DBInstanceIdentifier', 'Value': db_instance}]
        )
        
        write_iops = self.get_metric(
            namespace='AWS/RDS',
            metric_name='WriteIOPS',
            dimensions=[{'Name': 'DBInstanceIdentifier', 'Value': db_instance}]
        )
        
        print(f"Read IOPS: {read_iops:.2f if read_iops else 'N/A'}")
        print(f"Write IOPS: {write_iops:.2f if write_iops else 'N/A'}")
        
        # If using gp2 (3 IOPS per GB), max is limited
        if read_iops and read_iops > 3000:
            print("⚠️  High IOPS - may be hitting throughput limits")
    
    def check_alb_response_time(self, alb_name):
        """Check ALB response time"""
        print("\n[5] Application Load Balancer")
        print("-" * 50)
        
        response_time = self.get_metric(
            namespace='AWS/ApplicationELB',
            metric_name='TargetResponseTime',
            dimensions=[{'Name': 'LoadBalancer', 'Value': alb_name}]
        )
        
        if response_time is None:
            print("⚠️  Could not retrieve ALB metrics")
            return
        
        response_time_ms = response_time * 1000
        print(f"Target Response Time: {response_time_ms:.0f}ms")
        
        if response_time_ms > 1000:
            print("🔴 HIGH: ALB response time critical")
            self.issues_found.append(f"ALB response time high: {response_time_ms:.0f}ms")
        elif response_time_ms > 500:
            print("🟡 MODERATE: ALB response time elevated")
        else:
            print("✅ NORMAL: ALB response time healthy")
    
    def check_autoscaling(self, asg_name):
        """Check autoscaling group status"""
        print("\n[6] Autoscaling Group Status")
        print("-" * 50)
        
        try:
            response = self.asg_client.describe_auto_scaling_groups(
                AutoScalingGroupNames=[asg_name]
            )
            
            if not response['AutoScalingGroups']:
                print(f"⚠️  ASG '{asg_name}' not found")
                return
            
            asg = response['AutoScalingGroups'][0]
            desired = asg['DesiredCapacity']
            current = len(asg['Instances'])
            
            print(f"Desired Capacity: {desired}")
            print(f"Current Instances: {current}")
            
            if desired == current:
                print("✅ ASG is in sync")
            else:
                print(f"🟡 ASG scaling: {current}/{desired} instances ready")
                self.issues_found.append(f"ASG scaling in progress: {current}/{desired}")
                
        except ClientError as e:
            print(f"Error checking ASG: {e}")
    
    def check_recent_errors(self, log_group):
        """Check for recent errors in CloudWatch Logs"""
        print("\n[7] Recent Application Errors")
        print("-" * 50)
        
        try:
            query = """
            fields @timestamp, @message
            | filter @message like /ERROR|EXCEPTION|error/
            | stats count() as error_count
            """
            
            # This is a simplified check - use CloudWatch Logs Insights for detailed analysis
            response = self.logs_client.describe_log_streams(
                logGroupName=log_group,
                orderBy='LastEventTime',
                descending=True,
                limit=1
            )
            
            if response['logStreams']:
                print(f"✅ Log group '{log_group}' accessible")
            else:
                print(f"⚠️  No log streams found in '{log_group}'")
                
        except ClientError as e:
            print(f"Could not check logs: {e}")
    
    def generate_report(self):
        """Generate summary report"""
        print("\n" + "="*50)
        print("DIAGNOSTIC SUMMARY")
        print("="*50)
        
        if not self.issues_found:
            print("✅ No critical issues found!")
        else:
            print(f"🔴 {len(self.issues_found)} issue(s) detected:\n")
            for i, issue in enumerate(self.issues_found, 1):
                print(f"  {i}. {issue}")
        
        print("\nNext Steps:")
        print("1. Review CloudWatch Logs Insights for detailed analysis")
        print("2. Check RDS Performance Insights for slow queries")
        print("3. SSH to instances and run: top, jstack, jmap")
        print("4. Consider scaling up or optimizing code")

def main():
    import argparse
    
    parser = argparse.ArgumentParser(description='AWS Performance Diagnostics')
    parser.add_argument('--region', default='us-east-1', help='AWS region')
    parser.add_argument('--instance-id', required=True, help='EC2 instance ID')
    parser.add_argument('--db-instance', required=True, help='RDS instance identifier')
    parser.add_argument('--alb-name', required=True, help='ALB name (format: app/name/id)')
    parser.add_argument('--asg-name', required=True, help='Autoscaling group name')
    parser.add_argument('--log-group', help='CloudWatch log group name')
    
    args = parser.parse_args()
    
    diag = AWSPerformanceDiagnostics(region=args.region)
    
    print("🔍 Starting AWS Performance Diagnostics...")
    print(f"Region: {args.region}")
    print("="*50)
    
    diag.check_ec2_cpu(args.instance_id)
    diag.check_rds_connections(args.db_instance)
    diag.check_rds_cpu(args.db_instance)
    diag.check_rds_iops(args.db_instance)
    diag.check_alb_response_time(args.alb_name)
    diag.check_autoscaling(args.asg_name)
    
    if args.log_group:
        diag.check_recent_errors(args.log_group)
    
    diag.generate_report()

if __name__ == '__main__':
    main()
```

**Usage:**
```bash
python3 diagnose.py \
  --instance-id i-1234567890abcdef0 \
  --db-instance my-db \
  --alb-name app/my-alb/1234567890123456 \
  --asg-name my-asg \
  --log-group /aws/lambda/my-function
```

---

## Script 3: Real-time Monitoring Dashboard Script

Save as: `monitor.sh`

```bash
#!/bin/bash

# Real-time AWS performance monitoring
# Refreshes every 30 seconds

INSTANCE_ID=$1
DB_INSTANCE=$2
REGION=${3:-"us-east-1"}

while true; do
    clear
    echo "═══════════════════════════════════════════════════════════"
    echo "AWS Performance Monitor - $(date)"
    echo "═══════════════════════════════════════════════════════════"
    
    # EC2 Metrics
    CPU=$(aws cloudwatch get-metric-statistics \
      --namespace AWS/EC2 \
      --metric-name CPUUtilization \
      --dimensions Name=InstanceId,Value=$INSTANCE_ID \
      --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%S) \
      --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
      --period 300 \
      --statistics Average \
      --region $REGION \
      --query 'Datapoints[0].Average' \
      --output text 2>/dev/null)
    
    # RDS Metrics
    DB_CONN=$(aws cloudwatch get-metric-statistics \
      --namespace AWS/RDS \
      --metric-name DatabaseConnections \
      --dimensions Name=DBInstanceIdentifier,Value=$DB_INSTANCE \
      --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%S) \
      --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
      --period 300 \
      --statistics Maximum \
      --region $REGION \
      --query 'Datapoints[0].Maximum' \
      --output text 2>/dev/null)
    
    RDS_CPU=$(aws cloudwatch get-metric-statistics \
      --namespace AWS/RDS \
      --metric-name CPUUtilization \
      --dimensions Name=DBInstanceIdentifier,Value=$DB_INSTANCE \
      --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%S) \
      --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
      --period 300 \
      --statistics Average \
      --region $REGION \
      --query 'Datapoints[0].Average' \
      --output text 2>/dev/null)
    
    # Display
    echo ""
    echo "EC2 Instance: $INSTANCE_ID"
    echo "  CPU: ${CPU:--}%"
    echo ""
    echo "RDS Instance: $DB_INSTANCE"
    echo "  Connections: ${DB_CONN:--}"
    echo "  CPU: ${RDS_CPU:--}%"
    echo ""
    echo "═══════════════════════════════════════════════════════════"
    echo "Next refresh in 30 seconds... (Press Ctrl+C to exit)"
    
    sleep 30
done
```

**Usage:**
```bash
chmod +x monitor.sh
./monitor.sh i-1234567890abcdef0 my-db us-east-1
```

---

## Installation & Setup

### 1. Install AWS CLI
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

### 2. Configure AWS Credentials
```bash
aws configure
# Enter: AWS Access Key ID, Secret Access Key, Region, Output format
```

### 3. Install Python Dependencies (if using Python script)
```bash
pip install boto3
```

### 4. Run Diagnostics
```bash
# Bash version
./diagnose.sh

# Python version
python3 diagnose.py --instance-id ... --db-instance ... etc

# Continuous monitoring
./monitor.sh
```

---

## Tips for Effective Troubleshooting

1. **Schedule regular runs** to establish baseline
2. **Save output** for comparison: `./diagnose.sh > before.txt`
3. **Use with CloudWatch dashboard** for visual confirmation
4. **Check CloudTrail** for recent infrastructure changes
5. **Combine with manual inspection**: SSH to instance and run `top`, `jstack`

These scripts automate the first phase of diagnosis - metric collection. Always follow up with manual investigation based on findings.
