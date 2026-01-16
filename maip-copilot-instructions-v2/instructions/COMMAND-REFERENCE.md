# MAIP Command Reference

## Purpose

Quick lookup for all validated commands. Each command has been tested and documented with expected behavior.

**Trust these commands.** Only search for alternatives if a command fails with an unexpected error.

---

## AWS Profile Commands

### Validated Commands

```bash
# Switch environments (ALWAYS run one before AWS operations)
export AWS_PROFILE=maip-dev AWS_REGION=us-east-1    # Development
export AWS_PROFILE=maip-qa AWS_REGION=us-east-1     # QA
export AWS_PROFILE=maip-staging AWS_REGION=us-east-1 # Staging
export AWS_PROFILE=maip-prod AWS_REGION=us-east-1   # Production
export AWS_PROFILE=maip-prod AWS_REGION=us-west-2   # Production DR

# Verify identity (run after profile switch)
aws sts get-caller-identity
```

### Expected Output

```json
{
    "UserId": "AIDAXXXXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/your-username"
}
```

### Common Errors

| Error | Solution |
|-------|----------|
| `Unable to locate credentials` | Run `aws configure --profile maip-dev` |
| `ExpiredToken` | Re-authenticate with SSO or refresh credentials |

---

## EKS Commands

### Validated Commands

```bash
# Configure kubectl (run once per session)
aws eks update-kubeconfig --name maip-${MAIP_ENV:-dev}-eks-cluster

# List pods (most common)
kubectl get pods -n maip-services

# Tail logs
kubectl logs -f -l app={service-name} -n maip-services --tail=100

# Scale deployment
kubectl scale deployment {service-name} -n maip-services --replicas={count}

# Restart deployment
kubectl rollout restart deployment/{service-name} -n maip-services

# Check rollout status
kubectl rollout status deployment/{service-name} -n maip-services

# View HPA status
kubectl get hpa -n maip-services
```

### Service Names (Use Exactly)

```
product-catalog-service
customer-profile-service
merchant-application-service
pricing-engine-service
quote-service
order-service
notification-service
audit-service
```

---

## Kafka Commands

### Validated Commands

```bash
# Get brokers (run first, save to variable)
export MAIP_KAFKA_BROKERS=$(aws kafka get-bootstrap-brokers \
  --cluster-arn $(aws kafka list-clusters \
    --query "ClusterInfoList[?ClusterName=='maip-${MAIP_ENV:-dev}-kafka'].ClusterArn" \
    --output text) \
  --query 'BootstrapBrokerStringSasl' --output text)

# List topics
kafka-topics.sh --bootstrap-server $MAIP_KAFKA_BROKERS --list

# Check consumer lag (critical for debugging)
kafka-consumer-groups.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --describe --group maip-{service-name}-consumer

# Describe topic
kafka-topics.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --describe --topic maip.{domain}.{event}
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

## RDS Commands

### Validated Commands

```bash
# Get endpoint
RDS_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier maip-${MAIP_ENV:-dev}-postgres \
  --query 'DBInstances[0].Endpoint.Address' --output text)

# Connect
psql -h $RDS_ENDPOINT -U maip_admin -d maip_order

# Check status
aws rds describe-db-instances \
  --db-instance-identifier maip-${MAIP_ENV:-dev}-postgres \
  --query 'DBInstances[0].DBInstanceStatus'

# Create snapshot
aws rds create-db-snapshot \
  --db-instance-identifier maip-${MAIP_ENV}-postgres \
  --db-snapshot-identifier maip-${MAIP_ENV}-postgres-$(date +%Y%m%d-%H%M)
```

### Database Names

```
maip_product_catalog
maip_customer_profile
maip_merchant_application
maip_pricing_engine
maip_quote
maip_order
maip_notification
maip_audit
```

---

## DynamoDB Commands

### Validated Commands

```bash
# List tables
aws dynamodb list-tables --query "TableNames[?contains(@, 'maip-${MAIP_ENV:-dev}')]"

# Get item
aws dynamodb get-item \
  --table-name maip-${MAIP_ENV:-dev}-orders \
  --key '{"orderId": {"S": "order-123"}}'

# Query
aws dynamodb query \
  --table-name maip-${MAIP_ENV:-dev}-orders \
  --key-condition-expression "orderId = :id" \
  --expression-attribute-values '{":id": {"S": "order-123"}}'
```

### Table Names

```
maip-{env}-products
maip-{env}-customers
maip-{env}-applications
maip-{env}-pricing-rules
maip-{env}-quotes
maip-{env}-orders
maip-{env}-audit
maip-{env}-merchant-analytics
```

---

## EMR Commands

### Validated Commands

```bash
# Get cluster ID
export EMR_CLUSTER_ID=$(aws emr list-clusters --active \
  --query "Clusters[?Name=='maip-${MAIP_ENV}-emr'].Id" --output text)

# Submit Spark job
aws emr add-steps --cluster-id $EMR_CLUSTER_ID --steps '[{
  "Name": "MAIP ODS Sync",
  "Type": "Spark",
  "ActionOnFailure": "CONTINUE",
  "Args": [
    "--deploy-mode", "cluster",
    "s3://maip-'${MAIP_ENV}'-artifacts/spark/ods_sync.py",
    "--env", "'${MAIP_ENV}'"
  ]
}]'

# Check step status
aws emr list-steps --cluster-id $EMR_CLUSTER_ID --max-items 5
```

---

## Build Commands

### Validated Commands

```bash
# Build all microservices (from backend/microservices/)
./mvnw clean verify
# Time: ~8 minutes
# Expected: BUILD SUCCESS

# Build single service
./mvnw clean verify -pl {service}-impl -am
# Time: ~2 minutes

# Build with extra memory (if OutOfMemoryError)
MAVEN_OPTS="-Xmx2g" ./mvnw clean verify

# Skip tests (for quick iteration)
./mvnw clean package -DskipTests
```

### Required Order

1. `export AWS_PROFILE=maip-dev` (always first)
2. `cd DevOps/Local && docker-compose up -d` (if testing locally)
3. `./mvnw clean verify` (build)

---

## Test Commands

### Validated Commands

```bash
# Unit tests only
./mvnw test -pl {service}-impl

# Integration tests (requires local Docker)
./mvnw verify -pl {service}-impl -Pintegration

# Single test class
./mvnw test -pl {service}-impl -Dtest=OrderServiceTest

# Drools rules
./mvnw test -pl pricing-engine-rules -Dtest=PricingRulesTest
```

---

## Deploy Commands

### Validated Commands

```bash
# Deploy to EKS
kubectl apply -f DevOps/Kubernetes/services/{service}/ -n maip-services

# Check deployment
kubectl rollout status deployment/{service} -n maip-services

# Rollback
kubectl rollout undo deployment/{service} -n maip-services
```

---

## Local Development

### Validated Commands

```bash
# Start infrastructure (from DevOps/Local/)
docker-compose up -d
# Time: ~2 minutes
# Services: postgres, kafka, zookeeper, opensearch, prometheus, grafana

# Verify
docker-compose ps

# Stop
docker-compose down

# Stop and remove volumes (clean slate)
docker-compose down -v

# Run microservice locally
cd backend/microservices/{service}
./mvnw spring-boot:run -pl {service}-impl -Dspring.profiles.active=local

# Run Angular
cd ui/angular-app/maip-portal
npm install && npm start
# Opens at http://localhost:4200
```

---

## Emergency Commands

### High Kafka Lag

```bash
# 1. Check lag
kafka-consumer-groups.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --describe --group maip-{service}-consumer

# 2. Scale up
kubectl scale deployment {service} -n maip-services --replicas=12

# 3. Monitor
watch -n 5 'kubectl get pods -n maip-services -l app={service}'
```

### Service Unhealthy

```bash
# 1. Check pods
kubectl get pods -n maip-services -l app={service}

# 2. View logs
kubectl logs -f -l app={service} -n maip-services --tail=100

# 3. Restart
kubectl rollout restart deployment/{service} -n maip-services
```

### Database Connection Issues

```bash
# 1. Check RDS status
aws rds describe-db-instances --db-instance-identifier maip-${MAIP_ENV}-postgres

# 2. Check pod events
kubectl describe pod -l app={service} -n maip-services
```

---

## Trust These Commands

These commands have been validated. Only search for alternatives if:
- Command fails with unexpected error
- Resource names have changed
- New functionality not covered here

For detailed guides, see subdirectories: `aws/`, `microservices/`, `data-engineering/`, `deployment/`
