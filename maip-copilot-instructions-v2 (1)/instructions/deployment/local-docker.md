# Local Docker Development

## Purpose

Run MAIP infrastructure locally using Docker Compose for development and testing.

---

## Prerequisites

```bash
# Docker 24+
docker --version

# Docker Compose
docker-compose --version

# Verify Docker running
docker ps
```

---

## Start Infrastructure

```bash
cd DevOps/Local
docker-compose up -d
```

**Time:** ~2 minutes

---

## Verify Services

```bash
docker-compose ps
```

**Expected:** All services show "Up"

---

## Services Started

| Service | Port | Purpose |
|---------|------|---------|
| PostgreSQL | 5432 | Microservice databases |
| Zookeeper | 2181 | Kafka coordination |
| Kafka | 9092 | Event streaming |
| Schema Registry | 8081 | Avro schemas |
| OpenSearch | 9200 | Search & analytics |
| OpenSearch Dashboards | 5601 | Visualization |
| Prometheus | 9090 | Metrics |
| Grafana | 3000 | Dashboards |
| OTEL Collector | 4317 | Telemetry ingestion |
| LocalStack | 4566 | Mock AWS services |

---

## Initialize Databases

Run once after first start:

```bash
docker exec -it maip-postgres psql -U postgres -f /docker-entrypoint-initdb.d/init.sql
```

---

## Create Kafka Topics

```bash
# Core topics
docker exec -it maip-kafka kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --topic maip.product.created --partitions 3
docker exec -it maip-kafka kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --topic maip.customer.created --partitions 3
docker exec -it maip-kafka kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --topic maip.application.submitted --partitions 3
docker exec -it maip-kafka kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --topic maip.order.created --partitions 3
docker exec -it maip-kafka kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --topic maip.audit.events --partitions 3
```

---

## Run Microservices

Open separate terminals for each service:

```bash
# Terminal 1: Product Catalog
cd backend/microservices/product-catalog-service
./mvnw spring-boot:run -pl product-catalog-service-impl -Dspring.profiles.active=local

# Terminal 2: Customer Profile
cd backend/microservices/customer-profile-service
./mvnw spring-boot:run -pl customer-profile-service-impl -Dspring.profiles.active=local

# Terminal 3: Order Service
cd backend/microservices/order-service
./mvnw spring-boot:run -pl order-service-impl -Dspring.profiles.active=local
```

---

## Run Angular UI

```bash
cd ui/angular-app/maip-portal
npm install
npm start
```

Opens at: http://localhost:4200

---

## Access UIs

| UI | URL | Credentials |
|----|-----|-------------|
| Angular App | http://localhost:4200 | - |
| Grafana | http://localhost:3000 | admin/admin |
| Prometheus | http://localhost:9090 | - |
| OpenSearch Dashboards | http://localhost:5601 | - |

---

## Environment Variables

Set for local development:

```bash
export MAIP_ENV=local
export SPRING_PROFILES_ACTIVE=local
export KAFKA_BOOTSTRAP_SERVERS=localhost:9092
export SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5432/maip_order
```

---

## View Logs

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f kafka
docker-compose logs -f postgres
```

---

## Stop Services

```bash
# Stop (preserve data)
docker-compose down

# Stop and remove volumes (clean slate)
docker-compose down -v
```

---

## Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `Port already in use` | Previous container | `docker-compose down` then retry |
| `Container unhealthy` | Startup issue | Check logs: `docker-compose logs {service}` |
| `Connection refused` | Service not ready | Wait 30 seconds, retry |
| `No space left on device` | Docker disk full | `docker system prune -a` |

---

## Reset Everything

```bash
# Stop and remove all containers, volumes, networks
docker-compose down -v --remove-orphans

# Remove all MAIP images
docker images | grep maip | awk '{print $3}' | xargs docker rmi -f

# Start fresh
docker-compose up -d
```
