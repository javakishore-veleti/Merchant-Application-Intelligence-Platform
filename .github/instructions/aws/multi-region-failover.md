# Multi-Region Failover - MAIP

## Architecture

| Resource | Primary (us-east-1) | Secondary (us-west-2) |
|----------|---------------------|----------------------|
| EKS | maip-prod-eks-cluster | maip-prod-dr-eks-cluster |
| RDS | maip-prod-postgres (writer) | maip-prod-postgres-replica (read) |
| MSK | maip-prod-kafka | maip-prod-dr-kafka |
| DynamoDB | Global Table (auto-replicated) | Global Table (auto-replicated) |
| Route 53 | Primary | Failover |

## Health Check Status

```bash
# Check primary region
maip-prod
kubectl get nodes
aws rds describe-db-instances --db-instance-identifier maip-prod-postgres \
  --query 'DBInstances[0].DBInstanceStatus'

# Check secondary region
maip-prod-dr
kubectl get nodes
aws rds describe-db-instances --db-instance-identifier maip-prod-postgres-replica \
  --query 'DBInstances[0].DBInstanceStatus'
```

## Failover Procedures

### Step 1: Verify Primary is Down

```bash
maip-prod
kubectl get nodes  # Should timeout or show NotReady
aws rds describe-db-instances --db-instance-identifier maip-prod-postgres
```

### Step 2: Promote RDS Read Replica

```bash
maip-prod-dr
aws rds promote-read-replica \
  --db-instance-identifier maip-prod-postgres-replica

# Wait for promotion
aws rds wait db-instance-available \
  --db-instance-identifier maip-prod-postgres-replica
```

### Step 3: Update Route 53 (if not automatic)

```bash
# Get hosted zone ID
ZONE_ID=$(aws route53 list-hosted-zones \
  --query "HostedZones[?Name=='maip.example.com.'].Id" --output text)

# Update to point to DR region
aws route53 change-resource-record-sets \
  --hosted-zone-id $ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.maip.example.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "maip-prod-dr-alb.us-west-2.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

### Step 4: Scale DR EKS Cluster

```bash
maip-prod-dr

# Scale node group
aws eks update-nodegroup-config \
  --cluster-name maip-prod-dr-eks-cluster \
  --nodegroup-name maip-prod-dr-nodegroup \
  --scaling-config minSize=4,maxSize=20,desiredSize=8

# Scale deployments
kubectl scale deployment --all -n maip-services --replicas=4
```

### Step 5: Verify DR is Serving Traffic

```bash
# Check pods
kubectl get pods -n maip-services

# Check services
kubectl get svc -n maip-services

# Test endpoint
curl https://api.maip.example.com/actuator/health
```

## Failback Procedures

### Step 1: Restore Primary Infrastructure

```bash
maip-prod
# Rebuild or restore from backup
```

### Step 2: Resync Data

```bash
# DynamoDB: Already synced via Global Tables

# RDS: Create new replica from promoted instance
maip-prod-dr
aws rds create-db-instance-read-replica \
  --db-instance-identifier maip-prod-postgres-new \
  --source-db-instance-identifier maip-prod-postgres-replica \
  --source-region us-west-2 \
  --destination-region us-east-1
```

### Step 3: Gradual Traffic Shift

```bash
# Update Route 53 weights
# Start with 10% to primary, 90% to DR
# Monitor, then shift to 50/50, then 90/10, then 100/0
```

## DynamoDB Global Table Status

```bash
# Check replication status
aws dynamodb describe-table \
  --table-name maip-prod-merchant-analytics \
  --query 'Table.Replicas'

# Check replication lag (via CloudWatch)
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name ReplicationLatency \
  --dimensions Name=TableName,Value=maip-prod-merchant-analytics \
               Name=ReceivingRegion,Value=us-west-2 \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Average
```

## MSK Cross-Region Replication

```bash
# Check replicator status
aws kafka list-replicators \
  --query "Replicators[?contains(ReplicatorName, 'maip')].{Name:ReplicatorName,State:CurrentState}"
```
