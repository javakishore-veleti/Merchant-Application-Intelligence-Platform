# Flink Stream Processing - MAIP

## Overview
Java Flink jobs for real-time event processing, anomaly detection, and complex event processing (CEP).

## Location
`backend/data-services/flink-jobs/`

## Jobs

| Job | Purpose |
|-----|---------|
| StreamProcessor | Main event aggregation and routing |
| AnomalyDetector | Detect unusual patterns in transactions |
| CEPProcessor | Complex event pattern matching |

## OpenTelemetry
Service name: `maip-flink-stream-processor`

## Kafka Topics

**Consumed:**
- maip.product.*
- maip.customer.*
- maip.application.*
- maip.order.*

**Produced:**
- maip.analytics.aggregations
- maip.anomaly.detected

## Build

```bash
cd backend/data-services/flink-jobs
./mvnw clean package -DskipTests

# Output: target/maip-stream-processor.jar
```

## Run Locally (Docker)

```bash
# Start local Flink cluster
docker-compose -f docker-compose.flink.yaml up -d

# Submit job
docker exec -it flink-jobmanager flink run \
  /opt/flink/jobs/maip-stream-processor.jar \
  --env local \
  --kafka.bootstrap localhost:9092
```

## Run on EMR

```bash
# Upload JAR to S3
aws s3 cp target/maip-stream-processor.jar \
  s3://maip-${MAIP_ENV}-artifacts/flink/

# Submit job
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

## Code Structure

```java
public class StreamProcessor {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        
        // Kafka source
        KafkaSource<MaipEvent> source = KafkaSource.<MaipEvent>builder()
            .setBootstrapServers(kafkaBootstrap)
            .setTopics("maip.order.created", "maip.application.submitted")
            .setGroupId("maip-flink-processor")
            .setValueOnlyDeserializer(new MaipEventDeserializer())
            .build();
        
        DataStream<MaipEvent> events = env.fromSource(source, WatermarkStrategy.noWatermarks(), "Kafka");
        
        // Process and aggregate
        events
            .keyBy(MaipEvent::getCustomerId)
            .window(TumblingEventTimeWindows.of(Time.minutes(5)))
            .aggregate(new EventAggregator())
            .addSink(kafkaSink);
        
        env.execute("MAIP Stream Processor");
    }
}
```

## Anomaly Detection CEP Pattern

```java
Pattern<MaipEvent, ?> highVolumePattern = Pattern.<MaipEvent>begin("first")
    .where(event -> event.getEventType().equals("maip.order.created"))
    .timesOrMore(10)
    .within(Time.minutes(1));

CEP.pattern(orderStream, highVolumePattern)
    .select(matches -> new AnomalyEvent("HIGH_ORDER_VOLUME", matches));
```

## Monitoring

```bash
# Flink UI (via SSH tunnel)
ssh -L 8081:localhost:8081 hadoop@$MASTER_DNS
# Open http://localhost:8081

# Check running jobs
curl http://localhost:8081/jobs
```
