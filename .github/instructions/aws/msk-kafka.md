# MSK Kafka Management - MAIP

## Cluster Names
- `maip-dev-kafka`
- `maip-qa-kafka`
- `maip-staging-kafka`
- `maip-prod-kafka`

## Get Bootstrap Brokers

```bash
# Set environment variable for Kafka brokers
export MAIP_KAFKA_BROKERS=$(aws kafka get-bootstrap-brokers \
  --cluster-arn $(aws kafka list-clusters \
    --query "ClusterInfoList[?ClusterName=='maip-${MAIP_ENV:-dev}-kafka'].ClusterArn" \
    --output text) \
  --query 'BootstrapBrokerStringSasl' --output text)

echo $MAIP_KAFKA_BROKERS
```

## Topic Naming Convention
Pattern: `maip.{domain}.{event}`

| Domain | Topics |
|--------|--------|
| product | maip.product.created, maip.product.updated, maip.product.deleted |
| customer | maip.customer.created, maip.customer.updated, maip.customer.verified |
| application | maip.application.submitted, maip.application.approved, maip.application.rejected |
| pricing | maip.pricing.calculated, maip.pricing.rule-updated |
| quote | maip.quote.generated, maip.quote.accepted, maip.quote.expired |
| order | maip.order.created, maip.order.completed, maip.order.cancelled |
| audit | maip.audit.events |

## List Topics

```bash
kafka-topics.sh --bootstrap-server $MAIP_KAFKA_BROKERS --list
```

## Create Topic

```bash
kafka-topics.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --create --topic maip.{domain}.{event} \
  --partitions 6 --replication-factor 3
```

## Describe Topic

```bash
kafka-topics.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --describe --topic maip.order.created
```

## Consumer Groups

Consumer group naming: `maip-{service-name}-consumer`

```bash
# List all consumer groups
kafka-consumer-groups.sh --bootstrap-server $MAIP_KAFKA_BROKERS --list

# Describe specific consumer group
kafka-consumer-groups.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --describe --group maip-order-service-consumer

# Check lag for all groups
kafka-consumer-groups.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --describe --all-groups
```

## Check Consumer Lag (Emergency)

```bash
# Order service lag
kafka-consumer-groups.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --describe --group maip-order-service-consumer

# Merchant application service lag
kafka-consumer-groups.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --describe --group maip-merchant-application-consumer
```

## Reset Consumer Offset (USE WITH CAUTION)

```bash
# Stop consumers first, then:
kafka-consumer-groups.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --group maip-{service-name}-consumer \
  --topic maip.{domain}.{event} \
  --reset-offsets --to-latest --execute

# Or reset to specific offset
kafka-consumer-groups.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --group maip-{service-name}-consumer \
  --topic maip.{domain}.{event} \
  --reset-offsets --to-offset {offset} --execute
```

## Produce Test Message

```bash
kafka-console-producer.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --topic maip.{domain}.{event}
```

## Consume Messages (Debug)

```bash
kafka-console-consumer.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --topic maip.{domain}.{event} --from-beginning --max-messages 10
```
