# Flink Stream Processing

## Purpose

Java Flink jobs for real-time event processing, anomaly detection, and complex event processing (CEP).

---

## Location

`backend/data-services/flink-jobs/`

---

## Prerequisites

```bash
# Java 17
java -version

# Maven 3.9+
mvn -version
```

---

## Kafka Topics

### Consumed

- `maip.product.*`
- `maip.customer.*`
- `maip.application.*`
- `maip.order.*`

### Produced

- `maip.analytics.aggregations`
- `maip.anomaly.detected`

---

## Build Commands

```bash
cd backend/data-services/flink-jobs

# Build JAR
./mvnw clean package -DskipTests

# Output: target/maip-stream-processor.jar
```

---

## Run Locally

### Start Local Flink

```bash
docker-compose -f docker-compose.flink.yaml up -d
```

### Submit Job

```bash
docker exec -it flink-jobmanager flink run \
  /opt/flink/jobs/maip-stream-processor.jar \
  --env local \
  --kafka.bootstrap localhost:9092
```

### Check Jobs

```bash
curl http://localhost:8081/jobs
```

---

## Run on EMR

### Upload JAR

```bash
aws s3 cp target/maip-stream-processor.jar \
  s3://maip-${MAIP_ENV}-artifacts/flink/
```

### Submit Job

```bash
aws emr add-steps --cluster-id $EMR_CLUSTER_ID --steps '[{
  "Name": "MAIP Flink Stream Processor",
  "Type": "Custom",
  "ActionOnFailure": "CONTINUE",
  "Jar": "command-runner.jar",
  "Args": [
    "flink", "run", "-d",
    "s3://maip-'${MAIP_ENV}'-artifacts/flink/maip-stream-processor.jar",
    "--env", "'${MAIP_ENV}'",
    "--kafka.bootstrap", "'$MAIP_KAFKA_BROKERS'"
  ]
}]'
```

---

## Code Structure

### Main Class

```java
public class StreamProcessor {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = 
            StreamExecutionEnvironment.getExecutionEnvironment();
        
        KafkaSource<MaipEvent> source = KafkaSource.<MaipEvent>builder()
            .setBootstrapServers(kafkaBootstrap)
            .setTopics("maip.order.created", "maip.application.submitted")
            .setGroupId("maip-flink-processor")
            .setValueOnlyDeserializer(new MaipEventDeserializer())
            .build();
        
        DataStream<MaipEvent> events = env.fromSource(
            source, WatermarkStrategy.noWatermarks(), "Kafka");
        
        events
            .keyBy(MaipEvent::getCustomerId)
            .window(TumblingEventTimeWindows.of(Time.minutes(5)))
            .aggregate(new EventAggregator())
            .addSink(kafkaSink);
        
        env.execute("MAIP Stream Processor");
    }
}
```

### CEP Pattern (Anomaly Detection)

```java
Pattern<MaipEvent, ?> highVolumePattern = Pattern.<MaipEvent>begin("first")
    .where(event -> event.getEventType().equals("maip.order.created"))
    .timesOrMore(10)
    .within(Time.minutes(1));

CEP.pattern(orderStream, highVolumePattern)
    .select(matches -> new AnomalyEvent("HIGH_ORDER_VOLUME", matches));
```

---

## Test Commands

```bash
cd backend/data-services/flink-jobs

# Run tests
./mvnw test

# Run specific test
./mvnw test -Dtest=StreamProcessorTest
```

---

## Access Flink UI

### Local

```
http://localhost:8081
```

### EMR (via SSH Tunnel)

```bash
MASTER_DNS=$(aws emr describe-cluster --cluster-id $EMR_CLUSTER_ID \
  --query 'Cluster.MasterPublicDnsName' --output text)

ssh -i ~/.ssh/maip-emr-key.pem -L 8081:localhost:8081 hadoop@$MASTER_DNS
# Open http://localhost:8081
```

---

## Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `Kafka connection failed` | Wrong brokers | Verify `$MAIP_KAFKA_BROKERS` |
| `Checkpoint failed` | State backend issue | Check S3 permissions |
| `Backpressure` | Downstream slow | Scale up parallelism |
| `OutOfMemoryError` | State too large | Enable RocksDB state backend |

---

## Monitoring

### Check Backpressure

```bash
curl http://localhost:8081/jobs/{job-id}/vertices/{vertex-id}/backpressure
```

### Cancel Job

```bash
curl -X PATCH http://localhost:8081/jobs/{job-id}?mode=cancel
```
