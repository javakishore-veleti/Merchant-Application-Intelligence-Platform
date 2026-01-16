# Merchant Application Service - MAIP

## Overview
Handles merchant onboarding applications - submission, review, approval workflow.

## Details
- **Package:** `com.maip.merchantapplication`
- **Port:** 8083
- **Database:** `maip_{env}_merchant_application`
- **OpenTelemetry Name:** `maip-merchant-application-service`

## Kafka Topics
- maip.application.submitted (Producer)
- maip.application.approved (Producer)
- maip.application.rejected (Producer)
- maip.application.pending-review (Producer)
- maip.customer.verified (Consumer) - triggers application review

## API Endpoints
```
POST   /api/v1/applications              # Submit application
GET    /api/v1/applications/{id}         # Get by ID
GET    /api/v1/applications              # List (paginated)
PUT    /api/v1/applications/{id}         # Update application
POST   /api/v1/applications/{id}/approve # Approve application
POST   /api/v1/applications/{id}/reject  # Reject application
GET    /api/v1/applications/customer/{customerId}  # By customer
```

## Build & Deploy
```bash
cd backend/microservices/merchant-application-service
./mvnw clean verify
./mvnw spring-boot:run -pl merchant-application-service-impl -Dspring.profiles.active=local

# Deploy
kubectl apply -f DevOps/Kubernetes/services/merchant-application/ -n maip-services
```

## Emergency: High Backlog
```bash
# Check consumer lag
kafka-consumer-groups.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --describe --group maip-merchant-application-consumer

# Scale
kubectl scale deployment merchant-application-service -n maip-services --replicas=8
```
