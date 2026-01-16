# Product Catalog Service - MAIP

## Overview
Manages PoS terminal products, accessories, and bundles for merchant banking.

## Details
- **Package:** `com.maip.productcatalog`
- **Port:** 8081
- **Database:** `maip_{env}_product_catalog`
- **OpenTelemetry Name:** `maip-product-catalog-service`

## Kafka Topics

| Topic | Type | Description |
|-------|------|-------------|
| maip.product.created | Producer | New product added |
| maip.product.updated | Producer | Product details changed |
| maip.product.deleted | Producer | Product removed |

## API Endpoints

```
POST   /api/v1/products              # Create product
GET    /api/v1/products/{id}         # Get by ID
GET    /api/v1/products              # List (paginated)
PUT    /api/v1/products/{id}         # Update product
DELETE /api/v1/products/{id}         # Delete product
GET    /api/v1/products/category/{categoryId}  # By category
GET    /api/v1/products/search?q=    # Search products
```

## Build & Run

```bash
cd backend/microservices/product-catalog-service

# Build
./mvnw clean verify

# Run locally
./mvnw spring-boot:run -pl product-catalog-service-impl \
  -Dspring.profiles.active=local

# Build Docker image
docker build -t maip/product-catalog-service:latest .

# Run Docker
docker run -p 8081:8081 \
  -e SPRING_PROFILES_ACTIVE=local \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://host.docker.internal:5432/maip_product_catalog \
  maip/product-catalog-service:latest
```

## Deploy to EKS

```bash
kubectl apply -f DevOps/Kubernetes/services/product-catalog/ -n maip-services
kubectl rollout status deployment/product-catalog-service -n maip-services
```

## Check Logs

```bash
kubectl logs -f -l app=product-catalog-service -n maip-services --tail=100
```

## Scale

```bash
kubectl scale deployment product-catalog-service -n maip-services --replicas=4
```

## Key Entities

```java
@Entity
@Table(name = "products")
public class Product {
    UUID id;
    String sku;
    String name;
    String description;
    UUID categoryId;
    BigDecimal basePrice;
    ProductStatus status;  // ACTIVE, DISCONTINUED, COMING_SOON
    Instant createdAt;
    Instant updatedAt;
}

@Entity
@Table(name = "product_categories")
public class ProductCategory {
    UUID id;
    String name;
    String description;
    UUID parentCategoryId;
}
```

## Event Payload

```java
@Value @Builder
public class ProductCreatedEvent implements MaipEvent {
    String eventId = UUID.randomUUID().toString();
    String eventType = "maip.product.created";
    Instant timestamp = Instant.now();
    String correlationId;
    ProductPayload payload;
}

@Value @Builder
public class ProductPayload {
    UUID productId;
    String sku;
    String name;
    UUID categoryId;
    BigDecimal basePrice;
}
```
