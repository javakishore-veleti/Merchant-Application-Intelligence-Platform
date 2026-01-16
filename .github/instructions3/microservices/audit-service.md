# Audit Service - MAIP

## Overview
Captures and stores audit trail for all domain events across the platform.

## Details
- **Package:** `com.maip.audit`
- **Port:** 8088
- **Database:** DynamoDB `maip_{env}_audit`
- **OpenTelemetry Name:** `maip-audit-service`

## Kafka Topics (Consumer)
Subscribes to ALL maip.* topics for audit logging:
- maip.product.*
- maip.customer.*
- maip.application.*
- maip.pricing.*
- maip.quote.*
- maip.order.*
- maip.notification.*

## API Endpoints
```
GET /api/v1/audit/{entityType}/{entityId}  # Get audit trail for entity
GET /api/v1/audit/search?from=&to=&type=   # Search audit records
```

## Build & Deploy
```bash
cd backend/microservices/audit-service
./mvnw clean verify
./mvnw spring-boot:run -pl audit-service-impl -Dspring.profiles.active=local

# Deploy
kubectl apply -f DevOps/Kubernetes/services/audit-service/ -n maip-services
```

## DynamoDB Schema
- **Table:** `maip-{env}-audit`
- **PK:** `entityType#entityId`
- **SK:** `timestamp`
- **Attributes:** eventType, eventId, userId, payload, correlationId
