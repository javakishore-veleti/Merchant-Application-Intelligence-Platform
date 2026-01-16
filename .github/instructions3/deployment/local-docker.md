# Local Development - MAIP

## Prerequisites
- Docker & Docker Compose
- JDK 17
- Maven 3.9+
- Python 3.11+
- Node.js 18+ (for Angular)

## Start Infrastructure

```bash
cd DevOps/Local

# Start all infrastructure (Postgres, Kafka, OpenSearch, etc.)
docker-compose up -d

# Verify services
docker-compose ps
```

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

## Initialize Databases

```bash
# Create service databases
docker exec -it maip-postgres psql -U postgres -f /docker-entrypoint-initdb.d/init.sql
```

## Create Kafka Topics

```bash
docker exec -it maip-kafka kafka-topics.sh --bootstrap-server localhost:9092 --create --topic maip.product.created --partitions 3
docker exec -it maip-kafka kafka-topics.sh --bootstrap-server localhost:9092 --create --topic maip.customer.created --partitions 3
docker exec -it maip-kafka kafka-topics.sh --bootstrap-server localhost:9092 --create --topic maip.application.submitted --partitions 3
docker exec -it maip-kafka kafka-topics.sh --bootstrap-server localhost:9092 --create --topic maip.order.created --partitions 3
docker exec -it maip-kafka kafka-topics.sh --bootstrap-server localhost:9092 --create --topic maip.audit.events --partitions 3
```

## Run Microservices

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

# ... repeat for other services
```

## Run Angular UI

```bash
cd ui/angular-app/maip-portal
npm install
npm start
# Open http://localhost:4200
```

## Run Spark Locally

```bash
cd backend/data-services/spark-jobs
docker-compose -f docker-compose.spark.yaml up -d
python -m src.ods_sync.main --env local
```

## Access UIs

| UI | URL |
|----|-----|
| Angular App | http://localhost:4200 |
| Grafana | http://localhost:3000 (admin/admin) |
| Prometheus | http://localhost:9090 |
| OpenSearch Dashboards | http://localhost:5601 |

## Stop Everything

```bash
cd DevOps/Local
docker-compose down

# With volumes (clean slate)
docker-compose down -v
```

## Environment Variables

```bash
export MAIP_ENV=local
export SPRING_PROFILES_ACTIVE=local
export KAFKA_BOOTSTRAP_SERVERS=localhost:9092
export SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5432/maip_order
```
