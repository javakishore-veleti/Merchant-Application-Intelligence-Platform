# Merchant Application Intelligence Platform (MAIP)

[![AWS](https://img.shields.io/badge/AWS-Cloud%20Native-FF9900?logo=amazon-aws)](https://aws.amazon.com/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.2.6-6DB33F?logo=spring-boot)](https://spring.io/projects/spring-boot)
[![Kafka](https://img.shields.io/badge/Apache%20Kafka-Streaming-231F20?logo=apache-kafka)](https://kafka.apache.org/)
[![OpenTelemetry](https://img.shields.io/badge/OpenTelemetry-Observability-425CC7?logo=opentelemetry)](https://opentelemetry.io/)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

## Overview

**MAIP** is an enterprise-grade Merchant Banking Customer Onboarding Platform for Point-of-Sale (PoS) terminal sales. Built on the **Streaming Intelligence Framework (SIF)** principles, it combines event-driven microservices, real-time data pipelines, and LLMOps-enabled observability to deliver a secure, scalable, and intelligent merchant onboarding experience.

## Architecture

### System Architecture

```
                                    ┌──────────────────┐
                                    │   Angular UI     │
                                    │  (maip-portal)   │
                                    └────────┬─────────┘
                                             │
                                    ┌────────▼─────────┐
                                    │  AWS API Gateway │
                                    └────────┬─────────┘
                                             │
                                    ┌────────▼─────────┐
                                    │  Spring Cloud    │
                                    │  Gateway         │
                                    └────────┬─────────┘
                                             │
        ┌────────────────────────────────────┼────────────────────────────────────┐
        │                                    │                                    │
┌───────▼───────┐ ┌───────▼───────┐ ┌───────▼───────┐ ┌───────▼───────┐ ┌───────▼───────┐
│   Product     │ │   Customer    │ │   Merchant    │ │   Pricing     │ │    Quote      │
│   Catalog     │ │   Profile     │ │  Application  │ │   Engine      │ │   Service     │
│   Service     │ │   Service     │ │   Service     │ │  (Drools)     │ │               │
└───────┬───────┘ └───────┬───────┘ └───────┬───────┘ └───────┬───────┘ └───────┬───────┘
        │                 │                 │                 │                 │
        └─────────────────┴─────────────────┼─────────────────┴─────────────────┘
                                            │
                    ┌───────────────────────▼───────────────────────┐
                    │              Apache Kafka (MSK)                │
                    │    maip.product.* │ maip.customer.* │ etc.    │
                    └───────────────────────┬───────────────────────┘
                                            │
              ┌─────────────────────────────┼─────────────────────────────┐
              │                             │                             │
     ┌────────▼────────┐          ┌────────▼────────┐          ┌────────▼────────┐
     │   Order         │          │   Notification  │          │   Audit         │
     │   Service       │          │   Service       │          │   Service       │
     └────────┬────────┘          └─────────────────┘          └────────┬────────┘
              │                                                         │
              └─────────────────────────┬───────────────────────────────┘
                                        │
                    ┌───────────────────▼───────────────────────┐
                    │           Data Processing Layer            │
                    │  ┌─────────────┐      ┌─────────────┐     │
                    │  │ Apache Spark│      │Apache Flink │     │
                    │  │ (PySpark)   │      │   (Java)    │     │
                    │  │ ODS Sync    │      │ Stream CEP  │     │
                    │  └──────┬──────┘      └──────┬──────┘     │
                    └─────────┼────────────────────┼────────────┘
                              │                    │
                    ┌─────────▼────────────────────▼────────────┐
                    │         Operational Data Store (ODS)       │
                    │  ┌─────────────┐      ┌─────────────┐     │
                    │  │  DynamoDB   │      │   RDS       │     │
                    │  │  (Analytics)│      │ (PostgreSQL)│     │
                    │  └─────────────┘      └─────────────┘     │
                    └───────────────────────────────────────────┘
                                        │
                    ┌───────────────────▼───────────────────────┐
                    │              LLMOps Layer                  │
                    │  ┌─────────────┐      ┌─────────────┐     │
                    │  │ AWS Bedrock │      │  OpenSearch │     │
                    │  │ (Anomaly    │      │ (Vector DB) │     │
                    │  │  Detection) │      │             │     │
                    │  └─────────────┘      └─────────────┘     │
                    └───────────────────────────────────────────┘
```

## Business Domain

### Merchant Banking Customer Onboarding Flow

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Customer   │     │   Browse     │     │   Generate   │     │   Submit     │     │   Create     │
│   Registers  │────▶│   Products   │────▶│    Quote     │────▶│ Application  │────▶│    Order     │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
       │                    │                    │                    │                    │
       ▼                    ▼                    ▼                    ▼                    ▼
 maip.customer.       maip.product.        maip.quote.        maip.application.    maip.order.
   created              viewed              generated           submitted           created
```

### Quote Lifecycle Events

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Request    │     │   Price      │     │   Quote      │     │   Quote      │
│    Quote     │────▶│  Calculated  │────▶│  Generated   │────▶│   Sent       │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
       │                    │                    │                    │
       ▼                    ▼                    ▼                    ▼
 maip.quote.         maip.pricing.        maip.quote.         maip.quote.
  requested           calculated           generated             sent
                                                │
                    ┌───────────────────────────┼───────────────────────────┐
                    │                           │                           │
           ┌────────▼────────┐        ┌────────▼────────┐        ┌────────▼────────┐
           │     Quote       │        │     Quote       │        │     Quote       │
           │    Accepted     │        │    Rejected     │        │    Expired      │
           └────────┬────────┘        └─────────────────┘        └─────────────────┘
                    │
                    ▼
              maip.quote.accepted ──────▶ Triggers Order Creation
```

### AWS Infrastructure Management Events

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           AWS INFRASTRUCTURE EVENT FLOW                                  │
└─────────────────────────────────────────────────────────────────────────────────────────┘

EKS Cluster Events:
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Cluster    │     │  Node Group  │     │     Pod      │     │     HPA      │
│   Created    │────▶│    Scaled    │────▶│   Deployed   │────▶│   Triggered  │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
       │                    │                    │                    │
       ▼                    ▼                    ▼                    ▼
 maip.infra.eks.     maip.infra.eks.      maip.infra.eks.     maip.infra.eks.
 cluster.created     nodegroup.scaled     pod.deployed        hpa.triggered

MSK Kafka Events:
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│    Topic     │     │  Partition   │     │   Consumer   │     │  Consumer    │
│   Created    │────▶│   Expanded   │────▶│   Lag High   │────▶│  Rebalanced  │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
       │                    │                    │                    │
       ▼                    ▼                    ▼                    ▼
 maip.infra.msk.     maip.infra.msk.      maip.infra.msk.     maip.infra.msk.
 topic.created       partition.expanded   consumer.lag.high   consumer.rebalanced

RDS Database Events:
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Instance   │     │   Failover   │     │   Snapshot   │     │   Replica    │
│   Scaled     │────▶│   Initiated  │────▶│   Created    │────▶│   Promoted   │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
       │                    │                    │                    │
       ▼                    ▼                    ▼                    ▼
 maip.infra.rds.     maip.infra.rds.      maip.infra.rds.     maip.infra.rds.
 instance.scaled     failover.initiated   snapshot.created    replica.promoted

DynamoDB Events:
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│    Table     │     │   Capacity   │     │   Global     │     │   Backup     │
│   Created    │────▶│   Updated    │────▶│Table Synced  │────▶│   Created    │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
       │                    │                    │                    │
       ▼                    ▼                    ▼                    ▼
 maip.infra.ddb.     maip.infra.ddb.      maip.infra.ddb.     maip.infra.ddb.
 table.created       capacity.updated     globaltable.synced  backup.created

EMR Cluster Events:
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Cluster    │     │    Step      │     │   Cluster    │     │   Cluster    │
│   Started    │────▶│  Submitted   │────▶│    Scaled    │────▶│  Terminated  │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
       │                    │                    │                    │
       ▼                    ▼                    ▼                    ▼
 maip.infra.emr.     maip.infra.emr.      maip.infra.emr.     maip.infra.emr.
 cluster.started     step.submitted       cluster.scaled      cluster.terminated

Bedrock LLMOps Events:
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│    Model     │     │   Anomaly    │     │   Policy     │     │  Guardrail   │
│   Invoked    │────▶│   Detected   │────▶│  Violation   │────▶│   Triggered  │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
       │                    │                    │                    │
       ▼                    ▼                    ▼                    ▼
 maip.infra.bedrock. maip.infra.bedrock. maip.infra.bedrock. maip.infra.bedrock.
 model.invoked       anomaly.detected    policy.violation    guardrail.triggered
```

### Data Pipeline Events

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              DATA PIPELINE EVENT FLOW                                    │
└─────────────────────────────────────────────────────────────────────────────────────────┘

Spark Pipeline Events (Batch Processing):
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│     Job      │     │    Stage     │     │    Task      │     │     Job      │
│   Started    │────▶│   Completed  │────▶│    Failed    │────▶│  Completed   │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
       │                    │                    │                    │
       ▼                    ▼                    ▼                    ▼
 maip.pipeline.      maip.pipeline.       maip.pipeline.      maip.pipeline.
 spark.job.started   spark.stage.done     spark.task.failed   spark.job.completed

┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  ODS Sync    │     │   Records    │     │    Data      │     │  Analytics   │
│   Started    │────▶│  Processed   │────▶│  Validated   │────▶│   Ready      │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
       │                    │                    │                    │
       ▼                    ▼                    ▼                    ▼
 maip.pipeline.      maip.pipeline.       maip.pipeline.      maip.pipeline.
 spark.ods.started   spark.records.done   spark.data.valid    spark.analytics.ready

Flink Pipeline Events (Stream Processing):
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│     Job      │     │  Checkpoint  │     │  Watermark   │     │   Backpres-  │
│   Deployed   │────▶│   Completed  │────▶│   Advanced   │────▶│sure Detected │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
       │                    │                    │                    │
       ▼                    ▼                    ▼                    ▼
 maip.pipeline.      maip.pipeline.       maip.pipeline.      maip.pipeline.
 flink.job.deployed  flink.checkpoint.ok  flink.watermark.adv flink.backpressure

┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  CEP Pattern │     │   Window     │     │  Aggregation │     │   Alert      │
│   Matched    │────▶│   Triggered  │────▶│   Emitted    │────▶│  Generated   │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
       │                    │                    │                    │
       ▼                    ▼                    ▼                    ▼
 maip.pipeline.      maip.pipeline.       maip.pipeline.      maip.pipeline.
 flink.cep.matched   flink.window.fired   flink.agg.emitted   flink.alert.generated

Cross-Pipeline Events:
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Pipeline   │     │    Data      │     │   Schema     │     │   Lineage    │
│   Started    │────▶│   Quality    │────▶│   Evolved    │────▶│   Updated    │
│              │     │    Check     │     │              │     │              │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
       │                    │                    │                    │
       ▼                    ▼                    ▼                    ▼
 maip.pipeline.      maip.pipeline.       maip.pipeline.      maip.pipeline.
 started             data.quality.check   schema.evolved      lineage.updated
```

### Pricing Engine - Drools Rule Categories

| Rule Category | Description | Example |
|---------------|-------------|---------|
| **State Pricing** | Base pricing varies by state | California has different base than Texas |
| **State Tax** | Tax rates by jurisdiction | CA: 7.25%, TX: 6.25%, NY: 8% |
| **Customer Discounts** | Loyalty and volume discounts | Existing customers get 5% off |
| **Product Discounts** | Product-specific promotions | Terminal X is 10% off this quarter |
| **Bundle Discounts** | Multi-product purchases | Buy 3+ terminals, get 15% off |
| **Competitor Migration** | Discounts for switching | Own competitor PoS? Get 12% migration discount |

## Microservices

| Service | Port | Description | Database |
|---------|------|-------------|----------|
| **product-catalog-service** | 8081 | PoS terminals, accessories, bundles | PostgreSQL |
| **customer-profile-service** | 8082 | Merchant customer data, KYC | PostgreSQL |
| **merchant-application-service** | 8083 | Onboarding application workflow | PostgreSQL |
| **pricing-engine-service** | 8084 | Complex pricing with Drools rules | PostgreSQL |
| **quote-service** | 8085 | Price quote generation | PostgreSQL |
| **order-service** | 8086 | Order management and fulfillment | PostgreSQL |
| **notification-service** | 8087 | Email, SMS, push notifications | PostgreSQL |
| **audit-service** | 8088 | Audit trail for all events | DynamoDB |

## Technology Stack

### Backend
- **Java 17** (LTS)
- **Spring Boot 3.2.6**
- **Spring Cloud** (Gateway, Config, Service Discovery)
- **Spring Kafka** (Event-driven messaging)
- **Drools 8.x** (Business rules engine)
- **OpenTelemetry** (Distributed tracing)

### Data Engineering
- **Apache Kafka** (Event streaming via AWS MSK)
- **Apache Spark** (PySpark for batch processing)
- **Apache Flink** (Java for stream processing & CEP)

### Databases
- **PostgreSQL** (Transactional data via AWS RDS)
- **DynamoDB** (Operational data store, audit logs)
- **OpenSearch** (Search, analytics, vector DB)

### Frontend
- **Angular 17+** (Single Page Application)

### Cloud Infrastructure (AWS)
- **EKS** (Kubernetes orchestration)
- **MSK** (Managed Kafka)
- **RDS** (Managed PostgreSQL)
- **DynamoDB** (NoSQL with Global Tables)
- **EMR** (Managed Spark/Flink)
- **Bedrock** (LLM for anomaly detection)
- **OpenSearch Service** (Search and analytics)
- **CloudFormation** (Infrastructure as Code)

### Observability
- **OpenTelemetry** (Traces, metrics, logs)
- **Prometheus** (Metrics collection)
- **Grafana** (Visualization)
- **AWS CloudWatch** (Cloud-native monitoring)

## Repository Structure

```
merchant-application-intelligence-platform/
├── .github/
│   ├── copilot-instructions.md          # AI assistant context
│   └── workflows/                        # GitHub Actions CI/CD
├── instructions/                         # Modular Copilot instructions
│   ├── aws/                              # AWS infrastructure management
│   ├── microservices/                    # Service-specific guides
│   ├── data-engineering/                 # Spark/Flink operations
│   └── deployment/                       # Environment deployment
├── DevOps/
│   ├── Local/                            # Docker Compose for local dev
│   │   ├── docker-compose.yml
│   │   └── containers/
│   └── Cloud/
│       ├── AWS/cloudformation/           # CloudFormation templates
│       ├── Azure/                        # (Future)
│       └── GCP/                          # (Future)
├── backend/
│   ├── microservices/
│   │   ├── product-catalog-service/
│   │   ├── customer-profile-service/
│   │   ├── merchant-application-service/
│   │   ├── pricing-engine-service/
│   │   ├── quote-service/
│   │   ├── order-service/
│   │   ├── notification-service/
│   │   ├── audit-service/
│   │   ├── api-gateway/
│   │   └── shared-libraries/
│   └── data-services/
│       ├── spark-jobs/                   # PySpark ODS sync
│       └── flink-jobs/                   # Java stream processing
├── ui/
│   └── angular-app/
│       └── maip-portal/
├── llmops/
│   ├── anomaly-detection/
│   └── policy-compliance/
└── observability/
    ├── otel-config/
    ├── dashboards/
    └── alerts/
```

## Quick Start

### Prerequisites
- Docker & Docker Compose
- JDK 17
- Maven 3.9+
- Python 3.11+
- Node.js 18+
- AWS CLI v2

### Local Development

```bash
# Clone repository
git clone https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform.git
cd Merchant-Application-Intelligence-Platform

# Start infrastructure
cd DevOps/Local
docker-compose up -d

# Run a microservice
cd backend/microservices/order-service
./mvnw spring-boot:run -pl order-service-impl -Dspring.profiles.active=local

# Run Angular UI
cd ui/angular-app/maip-portal
npm install && npm start
```

### AWS Deployment

```bash
# Configure AWS profile
aws configure --profile maip-dev

# Set environment
export AWS_PROFILE=maip-dev AWS_REGION=us-east-1

# Deploy infrastructure
aws cloudformation deploy \
  --template-file DevOps/Cloud/AWS/cloudformation/networking/vpc.yaml \
  --stack-name maip-dev-networking \
  --capabilities CAPABILITY_IAM

# Configure kubectl
aws eks update-kubeconfig --name maip-dev-eks-cluster

# Deploy services
kubectl apply -f DevOps/Kubernetes/services/ -n maip-services
```

## GitHub Copilot Instructions

This repository uses modular GitHub Copilot instruction files to enable AI-assisted infrastructure management and code generation.

### Master Instructions
- [`.github/copilot-instructions.md`](.github/copilot-instructions.md) - Global context

### Modular Instructions
- [`instructions/aws/`](instructions/aws/) - AWS CLI commands for EKS, MSK, RDS, DynamoDB, EMR, Bedrock
- [`instructions/microservices/`](instructions/microservices/) - Service-specific build, deploy, debug commands
- [`instructions/data-engineering/`](instructions/data-engineering/) - Spark and Flink job operations
- [`instructions/deployment/`](instructions/deployment/) - Environment deployment procedures

### Usage in IDE

Open any instruction file in IntelliJ IDEA, VS Code, or PyCharm. When you type commands or ask questions, Copilot uses these instructions to generate context-aware suggestions with your actual resource names, AWS profiles, and deployment patterns.

## AWS Environments

| Environment | AWS Profile | Primary Region | DR Region | Purpose |
|-------------|-------------|----------------|-----------|---------|
| dev | `maip-dev` | us-east-1 | - | Development |
| qa | `maip-qa` | us-east-1 | - | Testing |
| staging | `maip-staging` | us-east-1 | us-west-2 | Pre-production |
| prod | `maip-prod` | us-east-1 | us-west-2 | Production |

## Key Metrics

| Metric | Baseline | With SIF | Improvement |
|--------|----------|----------|-------------|
| Processing Latency | 250ms | 155ms | ↓ 38% |
| Anomaly Detection Accuracy | 71% | 84% | ↑ 18% |
| Mean Time to Recovery | 12 min | 4.8 min | ↓ 60% |
| Observability Coverage | 65% | 93% | ↑ 43% |
| Policy Violations (monthly) | 14 | 4 | ↓ 71% |

## Event-Driven Architecture

### Kafka Topics

#### Business Domain Events
```
maip.product.created       maip.customer.created      maip.application.submitted
maip.product.updated       maip.customer.updated      maip.application.approved
maip.product.deleted       maip.customer.verified     maip.application.rejected
maip.product.viewed        maip.customer.deleted      maip.application.pending

maip.quote.requested       maip.order.created         maip.notification.sent
maip.quote.generated       maip.order.processing      maip.notification.failed
maip.quote.sent            maip.order.completed       maip.notification.delivered
maip.quote.accepted        maip.order.cancelled
maip.quote.rejected        maip.order.refunded        maip.audit.events
maip.quote.expired

maip.pricing.calculated    maip.pricing.rule-updated  maip.anomaly.detected
```

#### AWS Infrastructure Events
```
# EKS Events
maip.infra.eks.cluster.created       maip.infra.eks.nodegroup.scaled
maip.infra.eks.pod.deployed          maip.infra.eks.hpa.triggered
maip.infra.eks.deployment.failed     maip.infra.eks.service.unhealthy

# MSK Kafka Events
maip.infra.msk.topic.created         maip.infra.msk.partition.expanded
maip.infra.msk.consumer.lag.high     maip.infra.msk.consumer.rebalanced
maip.infra.msk.broker.unhealthy      maip.infra.msk.throughput.exceeded

# RDS Events
maip.infra.rds.instance.scaled       maip.infra.rds.failover.initiated
maip.infra.rds.snapshot.created      maip.infra.rds.replica.promoted
maip.infra.rds.connection.exhausted  maip.infra.rds.storage.threshold

# DynamoDB Events
maip.infra.ddb.table.created         maip.infra.ddb.capacity.updated
maip.infra.ddb.globaltable.synced    maip.infra.ddb.backup.created
maip.infra.ddb.throttle.detected     maip.infra.ddb.stream.enabled

# EMR Events
maip.infra.emr.cluster.started       maip.infra.emr.step.submitted
maip.infra.emr.cluster.scaled        maip.infra.emr.cluster.terminated
maip.infra.emr.step.failed           maip.infra.emr.spot.interrupted

# Bedrock LLMOps Events
maip.infra.bedrock.model.invoked     maip.infra.bedrock.anomaly.detected
maip.infra.bedrock.policy.violation  maip.infra.bedrock.guardrail.triggered
maip.infra.bedrock.quota.exceeded    maip.infra.bedrock.latency.high
```

#### Data Pipeline Events
```
# Spark Pipeline Events
maip.pipeline.spark.job.started      maip.pipeline.spark.job.completed
maip.pipeline.spark.stage.done       maip.pipeline.spark.task.failed
maip.pipeline.spark.ods.started      maip.pipeline.spark.ods.completed
maip.pipeline.spark.records.done     maip.pipeline.spark.data.valid
maip.pipeline.spark.analytics.ready  maip.pipeline.spark.shuffle.spill

# Flink Pipeline Events
maip.pipeline.flink.job.deployed     maip.pipeline.flink.job.failed
maip.pipeline.flink.checkpoint.ok    maip.pipeline.flink.checkpoint.failed
maip.pipeline.flink.watermark.adv    maip.pipeline.flink.backpressure
maip.pipeline.flink.cep.matched      maip.pipeline.flink.window.fired
maip.pipeline.flink.agg.emitted      maip.pipeline.flink.alert.generated

# Cross-Pipeline Events
maip.pipeline.started                maip.pipeline.completed
maip.pipeline.data.quality.check     maip.pipeline.data.quality.failed
maip.pipeline.schema.evolved         maip.pipeline.lineage.updated
maip.pipeline.sla.breached           maip.pipeline.retry.exhausted
```

### Event Envelope

```json
{
  "eventId": "uuid-v4",
  "eventType": "maip.order.created",
  "eventVersion": "1.0",
  "timestamp": "2026-01-16T12:00:00Z",
  "source": "order-service",
  "correlationId": "uuid-v4",
  "payload": { },
  "metadata": {
    "traceId": "string",
    "spanId": "string"
  }
}
```

## Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit changes (`git commit -m 'feat(order): add bulk order processing'`)
4. Push to branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

### Commit Convention
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation
- `refactor`: Code restructuring
- `test`: Adding tests
- `chore`: Maintenance

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## References

- IEEE ICCED 2025: *"Streaming Intelligence: Securing, Scaling and Observing Cloud-Native Data Pipelines with LLMOps"*
- [Spring Boot Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)

---

**Built with ❤️ for the Merchant Banking Industry**
