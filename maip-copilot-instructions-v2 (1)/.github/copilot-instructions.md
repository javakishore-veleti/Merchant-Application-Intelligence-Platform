# MAIP - Merchant Application Intelligence Platform

## Repository Summary

Enterprise-grade Merchant Banking Customer Onboarding Platform for Point-of-Sale (PoS) terminal sales. Built on the Streaming Intelligence Framework (SIF) with event-driven microservices, real-time data pipelines, and LLMOps-enabled observability.

**Size:** ~50,000 LOC | **Type:** Monorepo | **Languages:** Java 17, Python 3.11, TypeScript  
**Frameworks:** Spring Boot 3.2.6, Angular 17, PySpark, Apache Flink  
**Cloud:** AWS (EKS, MSK, RDS, DynamoDB, EMR, Bedrock, OpenSearch)

---

## Build & Validation Commands

### Prerequisites (Always Verify First)

```bash
java -version    # Requires: 17.x
mvn -version     # Requires: 3.9+
python --version # Requires: 3.11+
node -version    # Requires: 18+
aws --version    # Requires: 2.x
docker --version # Requires: 24+
```

### Bootstrap (Run Once After Clone)

```bash
# 1. Set AWS profile (ALWAYS do this first)
export AWS_PROFILE=maip-dev AWS_REGION=us-east-1

# 2. Start local infrastructure (takes ~2 minutes)
cd DevOps/Local && docker-compose up -d

# 3. Wait for services to be healthy
docker-compose ps  # All should show "Up"

# 4. Initialize databases
docker exec -it maip-postgres psql -U postgres -f /docker-entrypoint-initdb.d/init.sql
```

### Build Commands

| Component | Command | Working Directory | Time |
|-----------|---------|-------------------|------|
| All Microservices | `./mvnw clean verify` | `backend/microservices/` | ~8 min |
| Single Service | `./mvnw clean verify -pl {service}-impl -am` | `backend/microservices/{service}/` | ~2 min |
| Flink Jobs | `./mvnw clean package -DskipTests` | `backend/data-services/flink-jobs/` | ~3 min |
| Spark Jobs | `pip install -r requirements.txt` | `backend/data-services/spark-jobs/` | ~1 min |
| Angular UI | `npm install && npm run build` | `ui/angular-app/maip-portal/` | ~3 min |

### Test Commands

```bash
# Unit tests (ALWAYS run before committing)
./mvnw test -pl {service}-impl

# Integration tests (requires local Docker infrastructure)
./mvnw verify -pl {service}-impl -Pintegration

# Drools rule tests only
./mvnw test -pl pricing-engine-rules -Dtest=PricingRulesTest

# Python tests
cd backend/data-services/spark-jobs && pytest

# Angular tests
cd ui/angular-app/maip-portal && npm test
```

### Lint Commands

```bash
# Java (Checkstyle + SpotBugs)
./mvnw checkstyle:check spotbugs:check

# Python
cd backend/data-services/spark-jobs && flake8 src/ && black --check src/

# Angular
cd ui/angular-app/maip-portal && npm run lint
```

### Run Locally

```bash
# Single microservice
cd backend/microservices/{service}
./mvnw spring-boot:run -pl {service}-impl -Dspring.profiles.active=local

# Angular UI
cd ui/angular-app/maip-portal && npm start
# Opens at http://localhost:4200
```

---

## Project Layout

```
merchant-application-intelligence-platform/
├── .github/
│   ├── copilot-instructions.md      # THIS FILE - Agent instructions
│   └── workflows/                    # CI/CD pipelines
│       ├── ci.yml                    # PR validation
│       ├── cd-dev.yml                # Deploy to dev
│       └── cd-prod.yml               # Deploy to prod
├── instructions/                     # Detailed operation guides
│   ├── COMMAND-REFERENCE.md          # Quick command lookup
│   ├── aws/                          # AWS infrastructure commands
│   ├── microservices/                # Service-specific guides
│   ├── data-engineering/             # Spark/Flink operations
│   └── deployment/                   # Environment deployment
├── DevOps/
│   ├── Local/
│   │   └── docker-compose.yml        # Local infrastructure
│   └── Cloud/AWS/cloudformation/     # Infrastructure as Code
├── backend/
│   ├── microservices/
│   │   ├── product-catalog-service/      # Port 8081
│   │   ├── customer-profile-service/     # Port 8082
│   │   ├── merchant-application-service/ # Port 8083
│   │   ├── pricing-engine-service/       # Port 8084 (Drools rules)
│   │   ├── quote-service/                # Port 8085
│   │   ├── order-service/                # Port 8086
│   │   ├── notification-service/         # Port 8087
│   │   ├── audit-service/                # Port 8088
│   │   └── shared-libraries/             # Common code
│   └── data-services/
│       ├── spark-jobs/               # PySpark batch processing
│       └── flink-jobs/               # Java stream processing
├── ui/angular-app/maip-portal/       # Angular frontend
└── observability/                    # Dashboards, alerts
```

### Microservice Module Structure (Each Service)

```
{service}/
├── {service}-api/          # API interfaces, DTOs
├── {service}-common/       # Shared entities, constants
├── {service}-dao/          # JPA repositories, entities
├── {service}-service/      # Service interfaces
├── {service}-impl/         # Implementation, controllers
└── pom.xml                 # Parent POM
```

---

## CI/CD Validation Pipeline

### PR Checks (Must Pass Before Merge)

1. **Compile:** `./mvnw compile -DskipTests`
2. **Unit Tests:** `./mvnw test`
3. **Checkstyle:** `./mvnw checkstyle:check`
4. **SpotBugs:** `./mvnw spotbugs:check`
5. **Integration Tests:** `./mvnw verify -Pintegration`

### Pre-Commit Checklist

```bash
# ALWAYS run these before committing:
./mvnw clean verify           # Builds and tests
./mvnw checkstyle:check       # Code style
./mvnw spotbugs:check         # Static analysis
```

---

## Key Configuration Files

| File | Purpose |
|------|---------|
| `backend/microservices/pom.xml` | Parent POM, dependency versions |
| `backend/microservices/{service}/{service}-impl/src/main/resources/application.yml` | Service config |
| `backend/microservices/{service}/{service}-impl/src/main/resources/application-local.yml` | Local overrides |
| `pricing-engine-service/pricing-engine-rules/src/main/resources/rules/*.drl` | Drools pricing rules |
| `DevOps/Local/docker-compose.yml` | Local infrastructure |
| `DevOps/Cloud/AWS/cloudformation/` | AWS infrastructure templates |

---

## Common Errors & Workarounds

| Error | Cause | Solution |
|-------|-------|----------|
| `Connection refused: localhost:5432` | Postgres not running | `cd DevOps/Local && docker-compose up -d` |
| `Connection refused: localhost:9092` | Kafka not running | `cd DevOps/Local && docker-compose up -d` |
| `AWS credentials not found` | Profile not set | `export AWS_PROFILE=maip-dev` |
| `kubectl: command not found` | Not installed | `brew install kubectl` or use AWS CloudShell |
| `Tests fail with "no such table"` | DB not initialized | Run init.sql script (see Bootstrap) |
| `OutOfMemoryError during build` | Maven heap too small | `export MAVEN_OPTS="-Xmx2g"` |
| `npm ERR! peer dependency` | Node modules stale | `rm -rf node_modules && npm install` |

---

## AWS Resource Naming Convention

Pattern: `maip-{env}-{resource}`

| Resource | Dev | QA | Prod |
|----------|-----|-----|------|
| EKS Cluster | `maip-dev-eks-cluster` | `maip-qa-eks-cluster` | `maip-prod-eks-cluster` |
| RDS | `maip-dev-postgres` | `maip-qa-postgres` | `maip-prod-postgres` |
| MSK | `maip-dev-kafka` | `maip-qa-kafka` | `maip-prod-kafka` |
| DynamoDB | `maip-dev-{table}` | `maip-qa-{table}` | `maip-prod-{table}` |

---

## Kafka Topics

Pattern: `maip.{domain}.{event}`

Domains: `product`, `customer`, `application`, `pricing`, `quote`, `order`, `notification`, `audit`, `infra`, `pipeline`

---

## Trust These Instructions

**IMPORTANT:** Trust these instructions and only perform additional searches if:
- The documented command fails with an unexpected error
- The information appears incomplete for your specific task
- Files referenced do not exist at the documented paths

For detailed commands, see `instructions/COMMAND-REFERENCE.md`.
