# Merchant Application Intelligence Platform (MAIP)

[![AWS](https://img.shields.io/badge/AWS-Cloud%20Native-FF9900?logo=amazon-aws)](https://aws.amazon.com/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.2.6-6DB33F?logo=spring-boot)](https://spring.io/projects/spring-boot)
[![Kafka](https://img.shields.io/badge/Apache%20Kafka-Streaming-231F20?logo=apache-kafka)](https://kafka.apache.org/)
[![OpenTelemetry](https://img.shields.io/badge/OpenTelemetry-Observability-425CC7?logo=opentelemetry)](https://opentelemetry.io/)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

## Overview

**MAIP** is an enterprise-grade Merchant Banking Customer Onboarding Platform for Point-of-Sale (PoS) terminal sales. Built on the **Streaming Intelligence Framework (SIF)** principles, it combines event-driven microservices, real-time data pipelines, and LLMOps-enabled observability to deliver a secure, scalable, and intelligent merchant onboarding experience.

This implementation is based on the IEEE research paper: *"Streaming Intelligence: Securing, Scaling and Observing Cloud-Native Data Pipelines with LLMOps"* (IEEE ICCED 2025).

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

## Key Metrics (From SIF Research)

| Metric | Baseline | With SIF | Improvement |
|--------|----------|----------|-------------|
| Processing Latency | 250ms | 155ms | ↓ 38% |
| Anomaly Detection Accuracy | 71% | 84% | ↑ 18% |
| Mean Time to Recovery | 12 min | 4.8 min | ↓ 60% |
| Observability Coverage | 65% | 93% | ↑ 43% |
| Policy Violations (monthly) | 14 | 4 | ↓ 71% |

## Event-Driven Architecture

### Kafka Topics

```
maip.product.created       maip.customer.created      maip.application.submitted
maip.product.updated       maip.customer.updated      maip.application.approved
maip.product.deleted       maip.customer.verified     maip.application.rejected

maip.pricing.calculated    maip.quote.generated       maip.order.created
maip.pricing.rule-updated  maip.quote.accepted        maip.order.completed
                           maip.quote.expired         maip.order.cancelled

maip.notification.sent     maip.audit.events          maip.anomaly.detected
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