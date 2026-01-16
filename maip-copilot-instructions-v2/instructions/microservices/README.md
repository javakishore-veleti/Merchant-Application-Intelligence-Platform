# Microservices - Common Guide

## Purpose

Common build, test, and deployment commands for all MAIP microservices.

---

## Service Inventory

| Service | Port | Package | Database |
|---------|------|---------|----------|
| product-catalog-service | 8081 | `com.maip.productcatalog` | `maip_product_catalog` |
| customer-profile-service | 8082 | `com.maip.customerprofile` | `maip_customer_profile` |
| merchant-application-service | 8083 | `com.maip.merchantapplication` | `maip_merchant_application` |
| pricing-engine-service | 8084 | `com.maip.pricingengine` | `maip_pricing_engine` |
| quote-service | 8085 | `com.maip.quote` | `maip_quote` |
| order-service | 8086 | `com.maip.order` | `maip_order` |
| notification-service | 8087 | `com.maip.notification` | `maip_notification` |
| audit-service | 8088 | `com.maip.audit` | DynamoDB |

---

## Module Structure (All Services)

```
{service}/
├── {service}-api/          # API interfaces, DTOs, OpenAPI specs
├── {service}-common/       # Shared entities, enums, constants
├── {service}-dao/          # JPA repositories, entity definitions
├── {service}-service/      # Service interfaces
├── {service}-impl/         # Implementation, controllers, Kafka
└── pom.xml                 # Parent POM
```

---

## Build Commands

### Build All Microservices

```bash
cd backend/microservices
./mvnw clean verify
```

**Time:** ~8 minutes  
**Expected:** `BUILD SUCCESS`

### Build Single Service

```bash
cd backend/microservices/{service}
./mvnw clean verify -pl {service}-impl -am
```

**Time:** ~2 minutes

### Build Skip Tests (Quick Iteration)

```bash
./mvnw clean package -DskipTests
```

### Build with Extra Memory

```bash
MAVEN_OPTS="-Xmx2g" ./mvnw clean verify
```

---

## Test Commands

### Unit Tests

```bash
./mvnw test -pl {service}-impl
```

### Integration Tests (Requires Local Docker)

```bash
# Start infrastructure first
cd DevOps/Local && docker-compose up -d

# Run integration tests
cd backend/microservices/{service}
./mvnw verify -pl {service}-impl -Pintegration
```

### Single Test Class

```bash
./mvnw test -pl {service}-impl -Dtest=OrderServiceTest
```

### Single Test Method

```bash
./mvnw test -pl {service}-impl -Dtest=OrderServiceTest#testCreateOrder
```

---

## Run Locally

### Prerequisites

```bash
# Start infrastructure
cd DevOps/Local && docker-compose up -d

# Verify services running
docker-compose ps
```

### Run Service

```bash
cd backend/microservices/{service}
./mvnw spring-boot:run -pl {service}-impl -Dspring.profiles.active=local
```

### Run with Debug

```bash
./mvnw spring-boot:run -pl {service}-impl \
  -Dspring.profiles.active=local \
  -Dspring-boot.run.jvmArguments="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005"
```

---

## Deploy to EKS

### Build and Push Docker Image

```bash
cd backend/microservices/{service}

# Build image
docker build -t maip/{service}:${VERSION:-latest} .

# Tag for ECR
docker tag maip/{service}:${VERSION:-latest} \
  ${AWS_ACCOUNT}.dkr.ecr.us-east-1.amazonaws.com/maip/{service}:${VERSION:-latest}

# Login to ECR
aws ecr get-login-password | docker login --username AWS \
  --password-stdin ${AWS_ACCOUNT}.dkr.ecr.us-east-1.amazonaws.com

# Push
docker push ${AWS_ACCOUNT}.dkr.ecr.us-east-1.amazonaws.com/maip/{service}:${VERSION:-latest}
```

### Deploy to Kubernetes

```bash
kubectl apply -f DevOps/Kubernetes/services/{service}/ -n maip-services
kubectl rollout status deployment/{service} -n maip-services
```

### Update Image Only

```bash
kubectl set image deployment/{service} \
  {service}=${AWS_ACCOUNT}.dkr.ecr.us-east-1.amazonaws.com/maip/{service}:${VERSION} \
  -n maip-services
```

---

## Lint Commands

### Checkstyle

```bash
./mvnw checkstyle:check
```

### SpotBugs

```bash
./mvnw spotbugs:check
```

---

## Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `Connection refused: 5432` | Postgres not running | `cd DevOps/Local && docker-compose up -d` |
| `Connection refused: 9092` | Kafka not running | `cd DevOps/Local && docker-compose up -d` |
| `OutOfMemoryError` | Maven heap | `MAVEN_OPTS="-Xmx2g" ./mvnw ...` |
| `Test failure: no such table` | DB not initialized | Run init.sql script |
| `Port already in use` | Another service running | Kill process or use different port |

---

## Health Check Endpoints

All services expose:

```
GET /actuator/health     # Health status
GET /actuator/info       # Build info
GET /actuator/metrics    # Prometheus metrics
```

**Example:**
```bash
curl http://localhost:8086/actuator/health
```

---

## Kafka Consumer Groups

| Service | Consumer Group |
|---------|----------------|
| product-catalog-service | `maip-product-catalog-consumer` |
| customer-profile-service | `maip-customer-profile-consumer` |
| merchant-application-service | `maip-merchant-application-consumer` |
| pricing-engine-service | `maip-pricing-engine-consumer` |
| quote-service | `maip-quote-service-consumer` |
| order-service | `maip-order-service-consumer` |
| notification-service | `maip-notification-service-consumer` |
| audit-service | `maip-audit-service-consumer` |
