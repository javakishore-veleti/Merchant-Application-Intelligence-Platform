# RDS PostgreSQL Management

## Purpose

Manage Amazon RDS PostgreSQL instances for MAIP microservice databases.

---

## Prerequisites

```bash
# ALWAYS set profile first
export AWS_PROFILE=maip-${MAIP_ENV:-dev} AWS_REGION=us-east-1

# Get RDS endpoint
export RDS_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier maip-${MAIP_ENV:-dev}-postgres \
  --query 'DBInstances[0].Endpoint.Address' --output text)

echo $RDS_ENDPOINT  # Verify not empty
```

---

## Instance Names

| Environment | Instance Name |
|-------------|---------------|
| dev | `maip-dev-postgres` |
| qa | `maip-qa-postgres` |
| staging | `maip-staging-postgres` |
| prod | `maip-prod-postgres` |

---

## Database Names (One Per Service)

| Database | Service |
|----------|---------|
| `maip_product_catalog` | product-catalog-service |
| `maip_customer_profile` | customer-profile-service |
| `maip_merchant_application` | merchant-application-service |
| `maip_pricing_engine` | pricing-engine-service |
| `maip_quote` | quote-service |
| `maip_order` | order-service |
| `maip_notification` | notification-service |
| `maip_audit` | audit-service |

---

## Validated Commands

### Connect to Database

```bash
# Connect to specific database
psql -h $RDS_ENDPOINT -U maip_admin -d maip_order

# Connect with password prompt
PGPASSWORD=$DB_PASSWORD psql -h $RDS_ENDPOINT -U maip_admin -d maip_order
```

### Check Instance Status

```bash
# Get status
aws rds describe-db-instances \
  --db-instance-identifier maip-${MAIP_ENV:-dev}-postgres \
  --query 'DBInstances[0].DBInstanceStatus'
```

**Expected Output:** `"available"`

### Get Instance Details

```bash
aws rds describe-db-instances \
  --db-instance-identifier maip-${MAIP_ENV:-dev}-postgres \
  --query 'DBInstances[0].{Status:DBInstanceStatus,Class:DBInstanceClass,Engine:EngineVersion,Storage:AllocatedStorage}'
```

### Create Snapshot

```bash
aws rds create-db-snapshot \
  --db-instance-identifier maip-${MAIP_ENV}-postgres \
  --db-snapshot-identifier maip-${MAIP_ENV}-postgres-$(date +%Y%m%d-%H%M)
```

**Time:** ~5-15 minutes depending on database size

### Restore from Snapshot

```bash
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier maip-${MAIP_ENV}-postgres-restored \
  --db-snapshot-identifier {snapshot-id}
```

**Time:** ~15-30 minutes

### Modify Instance Class

```bash
aws rds modify-db-instance \
  --db-instance-identifier maip-${MAIP_ENV}-postgres \
  --db-instance-class db.r6g.xlarge \
  --apply-immediately
```

**Time:** ~10-20 minutes, causes brief outage

### Create Read Replica

```bash
aws rds create-db-instance-read-replica \
  --db-instance-identifier maip-${MAIP_ENV}-postgres-replica \
  --source-db-instance-identifier maip-${MAIP_ENV}-postgres \
  --db-instance-class db.r6g.large
```

**Time:** ~20-30 minutes

### CloudWatch Metrics

```bash
# CPU Utilization (last hour)
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=maip-${MAIP_ENV}-postgres \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Average

# Database Connections
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=maip-${MAIP_ENV}-postgres \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Maximum
```

---

## Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `Connection refused` | Wrong endpoint or SG | Verify `$RDS_ENDPOINT` and security groups |
| `password authentication failed` | Wrong credentials | Check Secrets Manager or env vars |
| `too many connections` | Connection pool exhausted | Restart pods or increase `max_connections` |
| `FATAL: database does not exist` | Wrong database name | Check `Database Names` table above |

---

## Emergency: Connection Pool Exhausted

```bash
# 1. Check current connections
PGPASSWORD=$DB_PASSWORD psql -h $RDS_ENDPOINT -U maip_admin -d postgres \
  -c "SELECT count(*) FROM pg_stat_activity;"

# 2. Kill idle connections
PGPASSWORD=$DB_PASSWORD psql -h $RDS_ENDPOINT -U maip_admin -d postgres \
  -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state = 'idle' AND query_start < now() - interval '10 minutes';"

# 3. Restart affected service
kubectl rollout restart deployment/{service} -n maip-services
```
