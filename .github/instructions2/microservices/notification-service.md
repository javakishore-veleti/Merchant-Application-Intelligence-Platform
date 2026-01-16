# Notification Service - MAIP

## Overview
Handles email, SMS, and push notifications for merchant onboarding events.

## Details
- **Package:** `com.maip.notification`
- **Port:** 8087
- **Database:** `maip_{env}_notification`
- **OpenTelemetry Name:** `maip-notification-service`

## Kafka Topics (Consumer)
- maip.application.submitted - Send confirmation
- maip.application.approved - Send approval notification
- maip.application.rejected - Send rejection with reasons
- maip.order.created - Send order confirmation
- maip.order.completed - Send fulfillment notification

## Kafka Topics (Producer)
- maip.notification.sent
- maip.notification.failed

## API Endpoints
```
POST   /api/v1/notifications/send          # Send notification
GET    /api/v1/notifications/{id}          # Get by ID
GET    /api/v1/notifications/customer/{customerId}  # By customer
```

## Build & Deploy
```bash
cd backend/microservices/notification-service
./mvnw clean verify
./mvnw spring-boot:run -pl notification-service-impl -Dspring.profiles.active=local

# Deploy
kubectl apply -f DevOps/Kubernetes/services/notification-service/ -n maip-services
```
