# MAIP Command Reference

## Purpose

Quick lookup for all validated commands. Each command has been tested and documented with expected behavior.

**Trust these commands.** Only search for alternatives if a command fails with an unexpected error.

---

## AWS Profile Commands

📁 [aws-profile-setup.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/aws/aws-profile-setup.md)

### Validated Commands

```bash
# Switch environments (ALWAYS run one before AWS operations)
export AWS_PROFILE=maip-dev AWS_REGION=us-east-1 MAIP_ENV=dev    # Development
export AWS_PROFILE=maip-qa AWS_REGION=us-east-1 MAIP_ENV=qa      # QA
export AWS_PROFILE=maip-staging AWS_REGION=us-east-1 MAIP_ENV=staging  # Staging
export AWS_PROFILE=maip-prod AWS_REGION=us-east-1 MAIP_ENV=prod  # Production
export AWS_PROFILE=maip-prod AWS_REGION=us-west-2 MAIP_ENV=prod  # Production DR

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

📁 [eks-cluster.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/aws/eks-cluster.md)

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

# Describe pod (debugging)
kubectl describe pod -l app={service-name} -n maip-services
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

📁 [msk-kafka.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/aws/msk-kafka.md)

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

# Check all consumer groups
kafka-consumer-groups.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --describe --all-groups

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

### Topic Pattern

`maip.{domain}.{event}` — Examples: `maip.order.created`, `maip.customer.verified`

---

## RDS Commands

📁 [rds-postgres.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/aws/rds-postgres.md)

### Validated Commands

```bash
# Get endpoint
export RDS_ENDPOINT=$(aws rds describe-db-instances \
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

## DynamoDB Commands

📁 [dynamodb.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/aws/dynamodb.md)

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

# Describe table
aws dynamodb describe-table --table-name maip-${MAIP_ENV:-dev}-orders \
  --query 'Table.{Status:TableStatus,ItemCount:ItemCount}'
```

### Table Names

`maip-{env}-products`, `maip-{env}-customers`, `maip-{env}-applications`, `maip-{env}-quotes`, `maip-{env}-orders`, `maip-{env}-audit`, `maip-{env}-merchant-analytics`

---

## EMR Commands

📁 [emr.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/aws/emr.md)

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

# Cancel step
aws emr cancel-steps --cluster-id $EMR_CLUSTER_ID --step-ids {step-id}
```

---

## CloudFormation Commands

📁 [cloudformation.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/aws/cloudformation.md)

### Validated Commands

```bash
# Deploy stack
aws cloudformation deploy \
  --template-file DevOps/Cloud/AWS/cloudformation/{category}/{template}.yaml \
  --stack-name maip-${MAIP_ENV}-{purpose} \
  --parameter-overrides file://parameters/${MAIP_ENV}/{purpose}.json \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM

# Check status
aws cloudformation describe-stacks \
  --stack-name maip-${MAIP_ENV}-{purpose} \
  --query 'Stacks[0].StackStatus'

# List MAIP stacks
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --query "StackSummaries[?contains(StackName, 'maip-${MAIP_ENV}')]"

# View events (debugging)
aws cloudformation describe-stack-events \
  --stack-name maip-${MAIP_ENV}-{purpose} --max-items 10
```

---

## Build Commands

📁 [microservices/README.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/microservices/README.md)

### Validated Commands

```bash
# Build all microservices (from backend/microservices/)
./mvnw clean verify
# Time: ~8 minutes | Expected: BUILD SUCCESS

# Build single service
./mvnw clean verify -pl {service}-impl -am
# Time: ~2 minutes

# Build with extra memory (if OutOfMemoryError)
MAVEN_OPTS="-Xmx2g" ./mvnw clean verify

# Skip tests (quick iteration)
./mvnw clean package -DskipTests

# Lint
./mvnw checkstyle:check spotbugs:check
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

# Single test method
./mvnw test -pl {service}-impl -Dtest=OrderServiceTest#testCreateOrder

# Drools rules
./mvnw test -pl pricing-engine-rules -Dtest=PricingRulesTest
```

---

## Deploy Commands

📁 [deployment/environments.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/deployment/environments.md)

### Validated Commands

```bash
# Deploy to EKS
kubectl apply -f DevOps/Kubernetes/services/{service}/ -n maip-services

# Check deployment
kubectl rollout status deployment/{service} -n maip-services

# Rollback
kubectl rollout undo deployment/{service} -n maip-services

# Update image only
kubectl set image deployment/{service} \
  {service}=${AWS_ACCOUNT}.dkr.ecr.us-east-1.amazonaws.com/maip/{service}:${VERSION} \
  -n maip-services
```

---

## Local Development

📁 [deployment/local-docker.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/deployment/local-docker.md)

### Validated Commands

```bash
# Start infrastructure (from DevOps/Local/)
docker-compose up -d
# Time: ~2 minutes

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

## Data Engineering

📁 [Spark](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/data-engineering/spark-ods-sync.md) | [Flink](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/data-engineering/flink-stream-processing.md)

### Spark Commands

```bash
# Run locally
cd backend/data-services/spark-jobs
python -m src.ods_sync.main --env local

# Submit to EMR
aws emr add-steps --cluster-id $EMR_CLUSTER_ID --steps '[...]'
```

### Flink Commands

```bash
# Build JAR
cd backend/data-services/flink-jobs
./mvnw clean package -DskipTests

# Run locally
docker exec -it flink-jobmanager flink run /opt/flink/jobs/maip-stream-processor.jar
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

## File Index

| Category | File | Link |
|----------|------|------|
| **AWS** | Profile Setup | [aws-profile-setup.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/aws/aws-profile-setup.md) |
| **AWS** | EKS Cluster | [eks-cluster.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/aws/eks-cluster.md) |
| **AWS** | MSK Kafka | [msk-kafka.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/aws/msk-kafka.md) |
| **AWS** | RDS PostgreSQL | [rds-postgres.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/aws/rds-postgres.md) |
| **AWS** | DynamoDB | [dynamodb.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/aws/dynamodb.md) |
| **AWS** | EMR | [emr.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/aws/emr.md) |
| **AWS** | Bedrock LLMOps | [bedrock-llmops.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/aws/bedrock-llmops.md) |
| **AWS** | CloudFormation | [cloudformation.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/aws/cloudformation.md) |
| **AWS** | Multi-Region Failover | [multi-region-failover.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/aws/multi-region-failover.md) |
| **Services** | Common Guide | [microservices/README.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/microservices/README.md) |
| **Services** | Pricing Engine | [pricing-engine-service.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/microservices/pricing-engine-service.md) |
| **Services** | Order Service | [order-service.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/microservices/order-service.md) |
| **Data** | Spark ODS Sync | [spark-ods-sync.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/data-engineering/spark-ods-sync.md) |
| **Data** | Flink Streaming | [flink-stream-processing.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/data-engineering/flink-stream-processing.md) |
| **Deploy** | Local Docker | [local-docker.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/deployment/local-docker.md) |
| **Deploy** | Environments | [environments.md](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/blob/main/instructions/deployment/environments.md) |

---

## Trust These Commands

These commands have been validated. Only search for alternatives if:
- Command fails with unexpected error
- Resource names have changed
- New functionality not covered here
