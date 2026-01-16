# MSK Kafka Management

## Purpose

Manage Apache Kafka clusters on Amazon MSK for MAIP event streaming.

---

## Prerequisites

```bash
# ALWAYS set profile first
export AWS_PROFILE=maip-${MAIP_ENV:-dev} AWS_REGION=us-east-1

# Get Kafka brokers (required for all operations)
export MAIP_KAFKA_BROKERS=$(aws kafka get-bootstrap-brokers \
  --cluster-arn $(aws kafka list-clusters \
    --query "ClusterInfoList[?ClusterName=='maip-${MAIP_ENV:-dev}-kafka'].ClusterArn" \
    --output text) \
  --query 'BootstrapBrokerStringSasl' --output text)

echo $MAIP_KAFKA_BROKERS  # Verify not empty
```

---

## Cluster Names

| Environment | Cluster Name |
|-------------|--------------|
| dev | `maip-dev-kafka` |
| qa | `maip-qa-kafka` |
| staging | `maip-staging-kafka` |
| prod | `maip-prod-kafka` |

---

## Validated Commands

### Topic Management

```bash
# List all topics
kafka-topics.sh --bootstrap-server $MAIP_KAFKA_BROKERS --list

# Create topic
kafka-topics.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --create --topic maip.{domain}.{event} \
  --partitions 6 --replication-factor 3

# Describe topic
kafka-topics.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --describe --topic maip.order.created
```

### Consumer Group Management (Most Important)

```bash
# List consumer groups
kafka-consumer-groups.sh --bootstrap-server $MAIP_KAFKA_BROKERS --list

# Check lag for specific group (CRITICAL for debugging)
kafka-consumer-groups.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --describe --group maip-{service-name}-consumer

# Check lag for all groups
kafka-consumer-groups.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --describe --all-groups
```

**Expected Output:**
```
GROUP                          TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
maip-order-service-consumer    maip.quote.accepted  0     1523            1523            0
maip-order-service-consumer    maip.quote.accepted  1     1456            1458            2
```

**LAG > 1000:** Service is falling behind. Scale up.

### Reset Consumer Offset (Use With Caution)

```bash
# Stop consumers first, then:

# Reset to latest (skip backlog)
kafka-consumer-groups.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --group maip-{service-name}-consumer \
  --topic maip.{domain}.{event} \
  --reset-offsets --to-latest --execute

# Reset to earliest (reprocess all)
kafka-consumer-groups.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --group maip-{service-name}-consumer \
  --topic maip.{domain}.{event} \
  --reset-offsets --to-earliest --execute
```

### Debug: Consume Messages

```bash
# Read messages (debug only)
kafka-console-consumer.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --topic maip.{domain}.{event} --from-beginning --max-messages 10
```

### Debug: Produce Test Message

```bash
# Send test message
kafka-console-producer.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --topic maip.{domain}.{event}
# Then type message and press Enter
```

---

## Topic Naming Convention

Pattern: `maip.{domain}.{event}`

### Business Domain Topics

```
maip.product.created        maip.product.updated        maip.product.deleted
maip.customer.created       maip.customer.updated       maip.customer.verified
maip.application.submitted  maip.application.approved   maip.application.rejected
maip.pricing.calculated     maip.pricing.rule-updated
maip.quote.requested        maip.quote.generated        maip.quote.accepted
maip.order.created          maip.order.completed        maip.order.cancelled
maip.notification.sent      maip.audit.events
```

### Consumer Group Names

```
maip-product-catalog-consumer
maip-customer-profile-consumer
maip-merchant-application-consumer
maip-pricing-engine-consumer
maip-quote-service-consumer
maip-order-service-consumer
maip-notification-service-consumer
maip-audit-service-consumer
```

---

## Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `Connection refused` | Wrong brokers or not set | Re-run `export MAIP_KAFKA_BROKERS=...` |
| `Topic not found` | Typo in topic name | Check `kafka-topics.sh --list` |
| `Not authorized` | IAM policy missing | Check AWS profile permissions |
| `Offset out of range` | Consumer offset invalid | Reset offset to latest/earliest |

---

## Emergency: High Consumer Lag

```bash
# 1. Check current lag
kafka-consumer-groups.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --describe --group maip-{service}-consumer

# 2. Scale up consumers
kubectl scale deployment {service} -n maip-services --replicas=12

# 3. Monitor lag decreasing
watch -n 10 'kafka-consumer-groups.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --describe --group maip-{service}-consumer'
```
