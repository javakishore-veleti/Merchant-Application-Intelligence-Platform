# MAIP Instructions Command Reference

Quick reference to all available commands organized by category. Each section links to detailed instruction files.

---

## AWS Infrastructure Commands

### AWS Profile Management
📁 [`instructions/aws/aws-profile-setup.md`](aws/aws-profile-setup.md)

| Command | Description |
|---------|-------------|
| `maip-dev` | Switch to development environment |
| `maip-qa` | Switch to QA environment |
| `maip-staging` | Switch to staging environment |
| `maip-prod` | Switch to production (us-east-1) |
| `maip-prod-dr` | Switch to production DR (us-west-2) |
| `maip-whoami` | Verify current AWS identity and region |

---

### EKS Cluster Operations
📁 [`instructions/aws/eks-cluster.md`](aws/eks-cluster.md)

| Command | Description |
|---------|-------------|
| `aws eks update-kubeconfig` | Configure kubectl for cluster |
| `kubectl get nodes` | List cluster nodes |
| `kubectl get pods -n maip-services` | List all MAIP pods |
| `kubectl scale deployment` | Scale a service up/down |
| `kubectl logs -f -l app={service}` | Tail service logs |
| `kubectl rollout restart` | Restart a deployment |
| `kubectl rollout status` | Check deployment status |
| `kubectl get hpa` | View horizontal pod autoscaler status |
| `aws eks update-nodegroup-config` | Scale EKS node group |

---

### MSK Kafka Operations
📁 [`instructions/aws/msk-kafka.md`](aws/msk-kafka.md)

| Command | Description |
|---------|-------------|
| `aws kafka get-bootstrap-brokers` | Get Kafka broker endpoints |
| `kafka-topics.sh --list` | List all topics |
| `kafka-topics.sh --create` | Create a new topic |
| `kafka-topics.sh --describe` | Describe topic configuration |
| `kafka-consumer-groups.sh --list` | List consumer groups |
| `kafka-consumer-groups.sh --describe` | Check consumer lag |
| `kafka-consumer-groups.sh --reset-offsets` | Reset consumer offset |
| `kafka-console-consumer.sh` | Consume messages (debug) |
| `kafka-console-producer.sh` | Produce test messages |

---

### RDS PostgreSQL Operations
📁 [`instructions/aws/rds-postgres.md`](aws/rds-postgres.md)

| Command | Description |
|---------|-------------|
| `aws rds describe-db-instances` | Get instance details |
| `psql -h $RDS_ENDPOINT` | Connect to database |
| `aws rds create-db-snapshot` | Create manual snapshot |
| `aws rds restore-db-instance-from-db-snapshot` | Restore from snapshot |
| `aws rds create-db-instance-read-replica` | Create read replica |
| `aws rds modify-db-instance` | Modify instance class |
| `aws cloudwatch get-metric-statistics` | Get RDS metrics |

---

### DynamoDB Operations
📁 [`instructions/aws/dynamodb.md`](aws/dynamodb.md)

| Command | Description |
|---------|-------------|
| `aws dynamodb list-tables` | List all MAIP tables |
| `aws dynamodb describe-table` | Get table details |
| `aws dynamodb get-item` | Retrieve single item |
| `aws dynamodb query` | Query by partition key |
| `aws dynamodb scan` | Scan table (use sparingly) |
| `aws dynamodb update-table` | Update capacity/settings |
| `aws dynamodb create-backup` | Create table backup |
| `aws dynamodb restore-table-from-backup` | Restore from backup |
| `aws dynamodb update-continuous-backups` | Enable PITR |

---

### EMR Spark/Flink Operations
📁 [`instructions/aws/emr.md`](aws/emr.md)

| Command | Description |
|---------|-------------|
| `aws emr list-clusters` | List active clusters |
| `aws emr describe-cluster` | Get cluster details |
| `aws emr add-steps` | Submit Spark/Flink job |
| `aws emr list-steps` | List job steps |
| `aws emr describe-step` | Get step status |
| `aws emr cancel-steps` | Cancel running step |
| `aws emr modify-instance-groups` | Scale cluster |
| `aws s3 cp` | Upload job artifacts to S3 |

---

### AWS Bedrock LLMOps
📁 [`instructions/aws/bedrock-llmops.md`](aws/bedrock-llmops.md)

| Command | Description |
|---------|-------------|
| `aws bedrock list-foundation-models` | List available models |
| `aws bedrock-runtime invoke-model` | Invoke LLM for inference |
| `aws bedrock create-guardrail` | Create content guardrail |
| `aws ce get-cost-and-usage` | Check Bedrock costs |

---

### CloudFormation Deployment
📁 [`instructions/aws/cloudformation.md`](aws/cloudformation.md)

| Command | Description |
|---------|-------------|
| `aws cloudformation deploy` | Deploy/update stack |
| `aws cloudformation describe-stacks` | Get stack status |
| `aws cloudformation list-stacks` | List all MAIP stacks |
| `aws cloudformation describe-stack-events` | View stack events |
| `aws cloudformation delete-stack` | Delete stack |
| `aws cloudformation validate-template` | Validate template |

---

### Multi-Region Failover
📁 [`instructions/aws/multi-region-failover.md`](aws/multi-region-failover.md)

| Command | Description |
|---------|-------------|
| `aws rds promote-read-replica` | Promote DR database |
| `aws route53 change-resource-record-sets` | Update DNS routing |
| `aws dynamodb describe-table --query Replicas` | Check global table sync |
| `aws kafka list-replicators` | Check MSK replication |

---

## Microservice Commands

### All Services - Common Commands
📁 [`instructions/microservices/*.md`](microservices/)

| Command | Description |
|---------|-------------|
| `./mvnw clean verify` | Build and test service |
| `./mvnw spring-boot:run -Dspring.profiles.active=local` | Run locally |
| `docker build -t maip/{service}:latest .` | Build Docker image |
| `kubectl apply -f DevOps/Kubernetes/services/{service}/` | Deploy to EKS |
| `kubectl rollout status deployment/{service}` | Check deployment |
| `kubectl logs -f -l app={service}` | View logs |
| `kubectl scale deployment {service} --replicas={n}` | Scale service |

---

### Product Catalog Service
📁 [`instructions/microservices/product-catalog-service.md`](microservices/product-catalog-service.md)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/products` | POST | Create product |
| `/api/v1/products/{id}` | GET | Get product by ID |
| `/api/v1/products` | GET | List products |
| `/api/v1/products/{id}` | PUT | Update product |
| `/api/v1/products/{id}` | DELETE | Delete product |

---

### Customer Profile Service
📁 [`instructions/microservices/customer-profile-service.md`](microservices/customer-profile-service.md)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/customers` | POST | Create customer |
| `/api/v1/customers/{id}` | GET | Get customer |
| `/api/v1/customers/{id}` | PUT | Update customer |
| `/api/v1/customers/{id}/verify` | POST | Verify KYC |

---

### Merchant Application Service
📁 [`instructions/microservices/merchant-application-service.md`](microservices/merchant-application-service.md)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/applications` | POST | Submit application |
| `/api/v1/applications/{id}` | GET | Get application |
| `/api/v1/applications/{id}/approve` | POST | Approve application |
| `/api/v1/applications/{id}/reject` | POST | Reject application |

---

### Pricing Engine Service (Drools)
📁 [`instructions/microservices/pricing-engine-service.md`](microservices/pricing-engine-service.md)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/pricing/calculate` | POST | Calculate price |
| `/api/v1/pricing/simulate` | POST | Simulate pricing |
| `/api/v1/pricing/rules` | GET | List active rules |

| Drools Rule File | Purpose |
|------------------|---------|
| `state-pricing.drl` | Base pricing by state |
| `state-tax.drl` | State tax rates |
| `customer-discount.drl` | Customer loyalty discounts |
| `product-discount.drl` | Product promotions |
| `bundle-discount.drl` | Multi-product discounts |
| `competitor-discount.drl` | Migration discounts |

---

### Quote Service
📁 [`instructions/microservices/quote-service.md`](microservices/quote-service.md)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/quotes` | POST | Generate quote |
| `/api/v1/quotes/{id}` | GET | Get quote |
| `/api/v1/quotes/{id}/accept` | POST | Accept quote |

---

### Order Service
📁 [`instructions/microservices/order-service.md`](microservices/order-service.md)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/orders` | POST | Create order |
| `/api/v1/orders/{id}` | GET | Get order |
| `/api/v1/orders/{id}/status` | PUT | Update status |
| `/api/v1/orders/{id}/cancel` | POST | Cancel order |

---

### Notification Service
📁 [`instructions/microservices/notification-service.md`](microservices/notification-service.md)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/notifications/send` | POST | Send notification |
| `/api/v1/notifications/{id}` | GET | Get notification status |

---

### Audit Service
📁 [`instructions/microservices/audit-service.md`](microservices/audit-service.md)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/audit/{entityType}/{entityId}` | GET | Get audit trail |
| `/api/v1/audit/search` | GET | Search audit records |

---

## Data Engineering Commands

### Spark ODS Sync Jobs
📁 [`instructions/data-engineering/spark-ods-sync.md`](data-engineering/spark-ods-sync.md)

| Command | Description |
|---------|-------------|
| `docker-compose -f docker-compose.spark.yaml up` | Start local Spark |
| `python -m src.ods_sync.main --env local` | Run ODS sync locally |
| `aws emr add-steps (Spark)` | Submit Spark job to EMR |
| `aws emr describe-step` | Check job status |
| `aws s3 cp` | Upload Spark artifacts |

---

### Flink Stream Processing
📁 [`instructions/data-engineering/flink-stream-processing.md`](data-engineering/flink-stream-processing.md)

| Command | Description |
|---------|-------------|
| `./mvnw clean package` | Build Flink job JAR |
| `docker-compose -f docker-compose.flink.yaml up` | Start local Flink |
| `flink run` | Submit job locally |
| `aws emr add-steps (Flink)` | Submit Flink job to EMR |
| `curl http://localhost:8081/jobs` | Check Flink jobs |

---

## Deployment Commands

### Local Docker Development
📁 [`instructions/deployment/local-docker.md`](deployment/local-docker.md)

| Command | Description |
|---------|-------------|
| `docker-compose up -d` | Start all infrastructure |
| `docker-compose ps` | Check service status |
| `docker-compose logs -f {service}` | View service logs |
| `docker-compose down` | Stop all services |
| `docker-compose down -v` | Stop and remove volumes |

---

### Environment Deployment
📁 [`instructions/deployment/environments.md`](deployment/environments.md)

| Command | Description |
|---------|-------------|
| `aws eks update-kubeconfig` | Configure kubectl |
| `kubectl apply -f` | Deploy Kubernetes manifests |
| `kubectl rollout status` | Verify deployment |
| `kubectl rollout undo` | Rollback deployment |
| `kubectl rollout history` | View deployment history |

---

## Quick Command Cheatsheet

### Emergency Response

```bash
# High Kafka consumer lag
kafka-consumer-groups.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --describe --group maip-{service}-consumer
kubectl scale deployment {service} -n maip-services --replicas=12

# Service unhealthy
kubectl get pods -n maip-services -l app={service}
kubectl logs -f -l app={service} -n maip-services --tail=100
kubectl rollout restart deployment/{service} -n maip-services

# Database connection issues
aws rds describe-db-instances --db-instance-identifier maip-{env}-postgres
kubectl describe pod -l app={service} -n maip-services

# Check all system health
kubectl get pods -n maip-services
kubectl get hpa -n maip-services
kafka-consumer-groups.sh --bootstrap-server $MAIP_KAFKA_BROKERS --describe --all-groups
```

### Daily Operations

```bash
# Morning health check
maip-prod
kubectl get pods -n maip-services
kubectl get hpa -n maip-services
aws rds describe-db-instances --query 'DBInstances[*].{ID:DBInstanceIdentifier,Status:DBInstanceStatus}'

# Deploy new version
kubectl set image deployment/{service} {service}=$ECR_REPO/{service}:$VERSION -n maip-services
kubectl rollout status deployment/{service} -n maip-services

# View application logs
kubectl logs -f -l app={service} -n maip-services --tail=100

# Check Kafka health
kafka-consumer-groups.sh --bootstrap-server $MAIP_KAFKA_BROKERS --describe --all-groups
```

---

## File Index

| Category | File | Purpose |
|----------|------|---------|
| **AWS** | `aws/aws-profile-setup.md` | Profile configuration |
| **AWS** | `aws/eks-cluster.md` | EKS management |
| **AWS** | `aws/msk-kafka.md` | Kafka operations |
| **AWS** | `aws/rds-postgres.md` | Database management |
| **AWS** | `aws/dynamodb.md` | DynamoDB operations |
| **AWS** | `aws/emr.md` | EMR Spark/Flink |
| **AWS** | `aws/bedrock-llmops.md` | LLM integration |
| **AWS** | `aws/cloudformation.md` | Infrastructure deployment |
| **AWS** | `aws/multi-region-failover.md` | DR procedures |
| **Microservices** | `microservices/product-catalog-service.md` | Product service |
| **Microservices** | `microservices/customer-profile-service.md` | Customer service |
| **Microservices** | `microservices/merchant-application-service.md` | Application service |
| **Microservices** | `microservices/pricing-engine-service.md` | Pricing with Drools |
| **Microservices** | `microservices/quote-service.md` | Quote service |
| **Microservices** | `microservices/order-service.md` | Order service |
| **Microservices** | `microservices/notification-service.md` | Notification service |
| **Microservices** | `microservices/audit-service.md` | Audit service |
| **Data** | `data-engineering/spark-ods-sync.md` | Spark batch jobs |
| **Data** | `data-engineering/flink-stream-processing.md` | Flink streaming |
| **Deploy** | `deployment/local-docker.md` | Local development |
| **Deploy** | `deployment/environments.md` | Environment deployment |
