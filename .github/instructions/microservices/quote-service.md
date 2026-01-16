# Quote Service - MAIP

## Overview
Generates and manages price quotes for merchant PoS product purchases.

## Details
- **Package:** `com.maip.quote`
- **Port:** 8085
- **Database:** `maip_{env}_quote`
- **OpenTelemetry Name:** `maip-quote-service`

## Kafka Topics
- maip.quote.generated (Producer)
- maip.quote.accepted (Producer)
- maip.quote.expired (Producer)
- maip.pricing.calculated (Consumer) - receives pricing data

## API Endpoints
```
POST   /api/v1/quotes              # Generate quote
GET    /api/v1/quotes/{id}         # Get by ID
GET    /api/v1/quotes              # List (paginated)
POST   /api/v1/quotes/{id}/accept  # Accept quote (creates order)
GET    /api/v1/quotes/customer/{customerId}  # By customer
```

## Build & Deploy
```bash
cd backend/microservices/quote-service
./mvnw clean verify
./mvnw spring-boot:run -pl quote-service-impl -Dspring.profiles.active=local

# Deploy
kubectl apply -f DevOps/Kubernetes/services/quote-service/ -n maip-services
```
