# DynamoDB Management - MAIP

## Table Naming
Pattern: `maip-{env}-{table-name}`

## Domain Tables

| Table | PK | SK | Purpose |
|-------|----|----|---------|
| maip-{env}-products | productId | version | Product catalog |
| maip-{env}-customers | customerId | timestamp | Customer profiles |
| maip-{env}-applications | applicationId | status#timestamp | Merchant applications |
| maip-{env}-pricing-rules | ruleId | effectiveDate | Pricing rules |
| maip-{env}-quotes | quoteId | customerId#timestamp | Quotes |
| maip-{env}-orders | orderId | customerId#timestamp | Orders |
| maip-{env}-audit | entityType#entityId | timestamp | Audit trail |
| maip-{env}-merchant-analytics | merchantId | eventType#timestamp | Consolidated analytics |

## List Tables

```bash
aws dynamodb list-tables --query "TableNames[?contains(@, 'maip-${MAIP_ENV:-dev}')]"
```

## Describe Table

```bash
aws dynamodb describe-table --table-name maip-${MAIP_ENV:-dev}-orders \
  --query 'Table.{Status:TableStatus,ItemCount:ItemCount,Size:TableSizeBytes}'
```

## Query Items

```bash
# Get order by ID
aws dynamodb get-item \
  --table-name maip-${MAIP_ENV:-dev}-orders \
  --key '{"orderId": {"S": "order-123"}}'

# Query by partition key
aws dynamodb query \
  --table-name maip-${MAIP_ENV:-dev}-orders \
  --key-condition-expression "orderId = :id" \
  --expression-attribute-values '{":id": {"S": "order-123"}}'
```

## Scan Table (Use Sparingly)

```bash
aws dynamodb scan \
  --table-name maip-${MAIP_ENV:-dev}-products \
  --max-items 10
```

## Enable Global Tables (Multi-Region)

```bash
# For production - replicate to us-west-2
maip-prod && aws dynamodb update-table \
  --table-name maip-prod-merchant-analytics \
  --replica-updates 'Create={RegionName=us-west-2}'
```

## Check Global Table Status

```bash
aws dynamodb describe-table \
  --table-name maip-prod-merchant-analytics \
  --query 'Table.Replicas'
```

## Backup Table

```bash
aws dynamodb create-backup \
  --table-name maip-${MAIP_ENV}-orders \
  --backup-name maip-${MAIP_ENV}-orders-$(date +%Y%m%d)
```

## List Backups

```bash
aws dynamodb list-backups \
  --table-name maip-${MAIP_ENV}-orders
```

## Restore from Backup

```bash
aws dynamodb restore-table-from-backup \
  --target-table-name maip-${MAIP_ENV}-orders-restored \
  --backup-arn {backup-arn}
```

## Update Capacity (Provisioned Mode)

```bash
aws dynamodb update-table \
  --table-name maip-${MAIP_ENV}-orders \
  --provisioned-throughput ReadCapacityUnits=100,WriteCapacityUnits=50
```

## Enable Point-in-Time Recovery

```bash
aws dynamodb update-continuous-backups \
  --table-name maip-${MAIP_ENV}-orders \
  --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true
```
