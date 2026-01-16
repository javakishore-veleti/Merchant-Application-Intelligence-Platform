# RDS PostgreSQL Management - MAIP

## Instance Names
- `maip-dev-postgres`
- `maip-qa-postgres`
- `maip-staging-postgres`
- `maip-prod-postgres`

## Get Connection Endpoint

```bash
# Get RDS endpoint
aws rds describe-db-instances \
  --db-instance-identifier maip-${MAIP_ENV:-dev}-postgres \
  --query 'DBInstances[0].Endpoint.Address' --output text
```

## Service Databases

| Database | Service |
|----------|---------|
| maip_product_catalog | product-catalog-service |
| maip_customer_profile | customer-profile-service |
| maip_merchant_application | merchant-application-service |
| maip_pricing_engine | pricing-engine-service |
| maip_quote | quote-service |
| maip_order | order-service |
| maip_notification | notification-service |
| maip_audit | audit-service |

## Connect via psql

```bash
# Get endpoint and connect
RDS_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier maip-${MAIP_ENV:-dev}-postgres \
  --query 'DBInstances[0].Endpoint.Address' --output text)

psql -h $RDS_ENDPOINT -U maip_admin -d maip_order
```

## Check Instance Status

```bash
aws rds describe-db-instances \
  --db-instance-identifier maip-${MAIP_ENV:-dev}-postgres \
  --query 'DBInstances[0].{Status:DBInstanceStatus,Class:DBInstanceClass,Engine:EngineVersion}'
```

## Performance Insights

```bash
# Check if enabled
aws rds describe-db-instances \
  --db-instance-identifier maip-${MAIP_ENV:-dev}-postgres \
  --query 'DBInstances[0].PerformanceInsightsEnabled'
```

## Create Read Replica (Staging/Prod)

```bash
maip-staging && aws rds create-db-instance-read-replica \
  --db-instance-identifier maip-staging-postgres-replica \
  --source-db-instance-identifier maip-staging-postgres \
  --db-instance-class db.r6g.large \
  --availability-zone us-west-2a
```

## Modify Instance Class

```bash
aws rds modify-db-instance \
  --db-instance-identifier maip-${MAIP_ENV}-postgres \
  --db-instance-class db.r6g.xlarge \
  --apply-immediately
```

## Create Snapshot

```bash
aws rds create-db-snapshot \
  --db-instance-identifier maip-${MAIP_ENV}-postgres \
  --db-snapshot-identifier maip-${MAIP_ENV}-postgres-$(date +%Y%m%d-%H%M)
```

## Restore from Snapshot

```bash
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier maip-${MAIP_ENV}-postgres-restored \
  --db-snapshot-identifier {snapshot-id}
```

## CloudWatch Metrics

```bash
# CPU Utilization
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=maip-${MAIP_ENV}-postgres \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Average
```
