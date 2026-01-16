# DynamoDB Management

## Purpose

Manage Amazon DynamoDB tables for MAIP operational data store and audit logs.

---

## Prerequisites

```bash
# ALWAYS set profile first
export AWS_PROFILE=maip-${MAIP_ENV:-dev} AWS_REGION=us-east-1
```

---

## Table Names

Pattern: `maip-{env}-{table-name}`

| Table | PK | SK | Purpose |
|-------|----|----|---------|
| `maip-{env}-products` | `productId` | `version` | Product catalog cache |
| `maip-{env}-customers` | `customerId` | `timestamp` | Customer profiles |
| `maip-{env}-applications` | `applicationId` | `status#timestamp` | Applications |
| `maip-{env}-pricing-rules` | `ruleId` | `effectiveDate` | Pricing rules |
| `maip-{env}-quotes` | `quoteId` | `customerId#timestamp` | Quotes |
| `maip-{env}-orders` | `orderId` | `customerId#timestamp` | Orders |
| `maip-{env}-audit` | `entityType#entityId` | `timestamp` | Audit trail |
| `maip-{env}-merchant-analytics` | `merchantId` | `eventType#timestamp` | Analytics |

---

## Validated Commands

### List Tables

```bash
aws dynamodb list-tables \
  --query "TableNames[?contains(@, 'maip-${MAIP_ENV:-dev}')]"
```

### Describe Table

```bash
aws dynamodb describe-table --table-name maip-${MAIP_ENV:-dev}-orders \
  --query 'Table.{Status:TableStatus,ItemCount:ItemCount,Size:TableSizeBytes}'
```

### Get Single Item

```bash
aws dynamodb get-item \
  --table-name maip-${MAIP_ENV:-dev}-orders \
  --key '{"orderId": {"S": "order-123"}}'
```

### Query by Partition Key

```bash
aws dynamodb query \
  --table-name maip-${MAIP_ENV:-dev}-orders \
  --key-condition-expression "orderId = :id" \
  --expression-attribute-values '{":id": {"S": "order-123"}}'
```

### Query with Sort Key Range

```bash
aws dynamodb query \
  --table-name maip-${MAIP_ENV:-dev}-audit \
  --key-condition-expression "entityType#entityId = :pk AND #ts BETWEEN :start AND :end" \
  --expression-attribute-names '{"#ts": "timestamp"}' \
  --expression-attribute-values '{
    ":pk": {"S": "order#order-123"},
    ":start": {"S": "2026-01-01T00:00:00Z"},
    ":end": {"S": "2026-01-31T23:59:59Z"}
  }'
```

### Scan Table (Use Sparingly)

```bash
aws dynamodb scan \
  --table-name maip-${MAIP_ENV:-dev}-products \
  --max-items 10
```

**Warning:** Scans are expensive. Use queries when possible.

### Put Item

```bash
aws dynamodb put-item \
  --table-name maip-${MAIP_ENV:-dev}-orders \
  --item '{
    "orderId": {"S": "order-456"},
    "customerId": {"S": "cust-789"},
    "status": {"S": "PENDING"},
    "createdAt": {"S": "2026-01-16T12:00:00Z"}
  }'
```

### Update Item

```bash
aws dynamodb update-item \
  --table-name maip-${MAIP_ENV:-dev}-orders \
  --key '{"orderId": {"S": "order-456"}}' \
  --update-expression "SET #s = :status" \
  --expression-attribute-names '{"#s": "status"}' \
  --expression-attribute-values '{":status": {"S": "COMPLETED"}}'
```

### Create Backup

```bash
aws dynamodb create-backup \
  --table-name maip-${MAIP_ENV}-orders \
  --backup-name maip-${MAIP_ENV}-orders-$(date +%Y%m%d)
```

### List Backups

```bash
aws dynamodb list-backups --table-name maip-${MAIP_ENV}-orders
```

### Enable Point-in-Time Recovery

```bash
aws dynamodb update-continuous-backups \
  --table-name maip-${MAIP_ENV}-orders \
  --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true
```

---

## Global Tables (Production Only)

### Check Replication Status

```bash
aws dynamodb describe-table \
  --table-name maip-prod-merchant-analytics \
  --query 'Table.Replicas'
```

### Add Replica Region

```bash
aws dynamodb update-table \
  --table-name maip-prod-merchant-analytics \
  --replica-updates 'Create={RegionName=us-west-2}'
```

**Time:** ~10-20 minutes

---

## Capacity Management

### Update Provisioned Capacity

```bash
aws dynamodb update-table \
  --table-name maip-${MAIP_ENV}-orders \
  --provisioned-throughput ReadCapacityUnits=100,WriteCapacityUnits=50
```

### Switch to On-Demand

```bash
aws dynamodb update-table \
  --table-name maip-${MAIP_ENV}-orders \
  --billing-mode PAY_PER_REQUEST
```

---

## Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `ResourceNotFoundException` | Table doesn't exist | Check table name: `aws dynamodb list-tables` |
| `ValidationException` | Wrong key format | Verify PK/SK structure in table schema |
| `ProvisionedThroughputExceededException` | Throttled | Switch to on-demand or increase capacity |
| `ConditionalCheckFailedException` | Condition not met | Check condition expression logic |

---

## Emergency: Throttling

```bash
# 1. Check current capacity
aws dynamodb describe-table --table-name maip-${MAIP_ENV}-orders \
  --query 'Table.ProvisionedThroughput'

# 2. Increase capacity
aws dynamodb update-table \
  --table-name maip-${MAIP_ENV}-orders \
  --provisioned-throughput ReadCapacityUnits=200,WriteCapacityUnits=100

# 3. Or switch to on-demand
aws dynamodb update-table \
  --table-name maip-${MAIP_ENV}-orders \
  --billing-mode PAY_PER_REQUEST
```
