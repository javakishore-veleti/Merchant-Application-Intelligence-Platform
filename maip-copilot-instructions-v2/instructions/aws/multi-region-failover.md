# Multi-Region Failover

## Purpose

Disaster recovery procedures for MAIP production environment failover from us-east-1 to us-west-2.

---

## Architecture

| Resource | Primary (us-east-1) | Secondary (us-west-2) |
|----------|---------------------|----------------------|
| EKS | `maip-prod-eks-cluster` | `maip-prod-dr-eks-cluster` |
| RDS | `maip-prod-postgres` (writer) | `maip-prod-postgres-replica` |
| MSK | `maip-prod-kafka` | `maip-prod-dr-kafka` |
| DynamoDB | Global Table (auto) | Global Table (auto) |
| Route 53 | Primary | Failover |

---

## Health Check Commands

### Check Primary Region

```bash
export AWS_PROFILE=maip-prod AWS_REGION=us-east-1

# EKS
aws eks update-kubeconfig --name maip-prod-eks-cluster
kubectl get nodes

# RDS
aws rds describe-db-instances --db-instance-identifier maip-prod-postgres \
  --query 'DBInstances[0].DBInstanceStatus'

# MSK
aws kafka describe-cluster --cluster-arn $(aws kafka list-clusters \
  --query "ClusterInfoList[?ClusterName=='maip-prod-kafka'].ClusterArn" --output text) \
  --query 'ClusterInfo.State'
```

### Check Secondary Region

```bash
export AWS_PROFILE=maip-prod AWS_REGION=us-west-2

# EKS
aws eks update-kubeconfig --name maip-prod-dr-eks-cluster
kubectl get nodes

# RDS Replica
aws rds describe-db-instances --db-instance-identifier maip-prod-postgres-replica \
  --query 'DBInstances[0].DBInstanceStatus'
```

---

## Failover Procedure

### Step 1: Verify Primary is Down

```bash
export AWS_REGION=us-east-1
kubectl get nodes  # Should timeout or show NotReady
```

### Step 2: Promote RDS Read Replica

```bash
export AWS_REGION=us-west-2

aws rds promote-read-replica \
  --db-instance-identifier maip-prod-postgres-replica

# Wait for promotion (5-10 minutes)
aws rds wait db-instance-available \
  --db-instance-identifier maip-prod-postgres-replica
```

**Validation:**
```bash
aws rds describe-db-instances --db-instance-identifier maip-prod-postgres-replica \
  --query 'DBInstances[0].{Status:DBInstanceStatus,Role:ReadReplicaSourceDBInstanceIdentifier}'
```

### Step 3: Update Route 53

```bash
ZONE_ID=$(aws route53 list-hosted-zones \
  --query "HostedZones[?Name=='maip.example.com.'].Id" --output text)

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
export AWS_REGION=us-west-2
aws eks update-kubeconfig --name maip-prod-dr-eks-cluster

# Scale node group
aws eks update-nodegroup-config \
  --cluster-name maip-prod-dr-eks-cluster \
  --nodegroup-name maip-prod-dr-nodegroup \
  --scaling-config minSize=4,maxSize=20,desiredSize=8

# Scale all deployments
kubectl scale deployment --all -n maip-services --replicas=4
```

### Step 5: Verify DR is Serving Traffic

```bash
# Check pods
kubectl get pods -n maip-services

# Test endpoint
curl -I https://api.maip.example.com/actuator/health
```

**Expected:** HTTP 200

---

## DynamoDB Global Table Status

DynamoDB Global Tables automatically replicate. Verify:

```bash
aws dynamodb describe-table \
  --table-name maip-prod-merchant-analytics \
  --query 'Table.Replicas'
```

### Check Replication Lag

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name ReplicationLatency \
  --dimensions Name=TableName,Value=maip-prod-merchant-analytics \
               Name=ReceivingRegion,Value=us-west-2 \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Average
```

---

## Failback Procedure

### Step 1: Restore Primary Infrastructure

Rebuild or restore us-east-1 resources.

### Step 2: Resync RDS

```bash
# Create new replica from promoted DR database
export AWS_REGION=us-west-2

aws rds create-db-instance-read-replica \
  --db-instance-identifier maip-prod-postgres-new \
  --source-db-instance-identifier maip-prod-postgres-replica \
  --source-region us-west-2 \
  --destination-region us-east-1
```

### Step 3: Gradual Traffic Shift

Update Route 53 weights: 10% → 50% → 90% → 100% to primary.

---

## Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `Replica already promoted` | Already a standalone | Use as new primary |
| `DNS not propagating` | TTL cache | Wait or lower TTL in advance |
| `Pods not scheduling` | Node capacity | Scale node group first |
| `DynamoDB replication lag` | High write volume | Monitor, usually self-corrects |

---

## Runbook Checklist

- [ ] Verify primary is down
- [ ] Promote RDS replica
- [ ] Wait for RDS to be available
- [ ] Update Route 53
- [ ] Scale DR EKS nodes
- [ ] Scale DR deployments
- [ ] Verify services healthy
- [ ] Verify API responding
- [ ] Notify stakeholders
