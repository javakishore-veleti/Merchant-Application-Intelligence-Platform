# Customer Profile Service - MAIP

## Overview
Manages merchant customer data, KYC information, and contact details.

## Details
- **Package:** `com.maip.customerprofile`
- **Port:** 8082
- **Database:** `maip_{env}_customer_profile`
- **OpenTelemetry Name:** `maip-customer-profile-service`

## Kafka Topics
- maip.customer.created (Producer)
- maip.customer.updated (Producer)
- maip.customer.verified (Producer)

## API Endpoints
```
POST   /api/v1/customers              # Create customer
GET    /api/v1/customers/{id}         # Get by ID
PUT    /api/v1/customers/{id}         # Update customer
GET    /api/v1/customers/search?q=    # Search customers
POST   /api/v1/customers/{id}/verify  # Verify KYC
```

## Build & Deploy
```bash
cd backend/microservices/customer-profile-service
./mvnw clean verify
./mvnw spring-boot:run -pl customer-profile-service-impl -Dspring.profiles.active=local

# Deploy
kubectl apply -f DevOps/Kubernetes/services/customer-profile/ -n maip-services
```
