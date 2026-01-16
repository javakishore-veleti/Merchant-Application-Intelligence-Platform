# Order Service

## Purpose

Manages order creation, processing, and fulfillment for merchant PoS purchases.

---

## Service Details

| Property | Value |
|----------|-------|
| Port | 8086 |
| Package | `com.maip.order` |
| Database | `maip_order` |
| OpenTelemetry | `maip-order-service` |

---

## Kafka Topics

| Topic | Type | Description |
|-------|------|-------------|
| `maip.order.created` | Producer | New order placed |
| `maip.order.completed` | Producer | Order fulfilled |
| `maip.order.cancelled` | Producer | Order cancelled |
| `maip.quote.accepted` | Consumer | Quote accepted, create order |
| `maip.application.approved` | Consumer | Application approved |

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/orders` | Create order |
| GET | `/api/v1/orders/{id}` | Get order by ID |
| GET | `/api/v1/orders` | List orders (paginated) |
| PUT | `/api/v1/orders/{id}/status` | Update order status |
| POST | `/api/v1/orders/{id}/cancel` | Cancel order |
| GET | `/api/v1/orders/customer/{customerId}` | Orders by customer |

---

## Build Commands

```bash
cd backend/microservices/order-service

# Build
./mvnw clean verify

# Build single module
./mvnw clean verify -pl order-service-impl -am
```

---

## Run Locally

```bash
./mvnw spring-boot:run -pl order-service-impl -Dspring.profiles.active=local
```

---

## Deploy to EKS

```bash
kubectl apply -f DevOps/Kubernetes/services/order-service/ -n maip-services
kubectl rollout status deployment/order-service -n maip-services
```

---

## Emergency: High Kafka Lag

```bash
# 1. Check lag
kafka-consumer-groups.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --describe --group maip-order-service-consumer

# 2. Scale up
kubectl scale deployment order-service -n maip-services --replicas=12

# 3. Monitor recovery
watch -n 5 'kubectl get pods -n maip-services -l app=order-service'
```

---

## Key Entities

### Order

```java
@Entity
@Table(name = "orders")
public class Order {
    UUID id;
    UUID customerId;
    UUID quoteId;
    OrderStatus status;  // PENDING, PROCESSING, COMPLETED, CANCELLED
    BigDecimal subtotal;
    BigDecimal taxAmount;
    BigDecimal discountAmount;
    BigDecimal totalAmount;
    String shippingAddress;
    Instant createdAt;
    Instant completedAt;
}
```

### OrderItem

```java
@Entity
@Table(name = "order_items")
public class OrderItem {
    UUID id;
    UUID orderId;
    UUID productId;
    int quantity;
    BigDecimal unitPrice;
    BigDecimal lineTotal;
}
```

---

## Event Payload

```java
@Value @Builder
public class OrderCreatedEvent implements MaipEvent {
    String eventId = UUID.randomUUID().toString();
    String eventType = "maip.order.created";
    Instant timestamp = Instant.now();
    String correlationId;
    OrderPayload payload;
}
```

---

## Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `Quote not found` | Invalid quoteId | Verify quote exists and is accepted |
| `Customer not approved` | Application not approved | Check application status |
| `Duplicate order` | Idempotency issue | Check existing orders for quoteId |
