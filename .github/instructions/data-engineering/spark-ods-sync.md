# Spark ODS Sync Job - MAIP

## Overview
PySpark job that consolidates data from domain-specific DynamoDB tables into the unified `merchant-analytics` table for cross-domain reporting.

## Location
`backend/data-services/spark-jobs/src/ods_sync/`

## Source Tables
- maip-{env}-products
- maip-{env}-customers
- maip-{env}-applications
- maip-{env}-quotes
- maip-{env}-orders

## Target Table
- maip-{env}-merchant-analytics

## OpenTelemetry
Service name: `maip-spark-ods-sync`

## Run Locally (Docker)

```bash
cd backend/data-services/spark-jobs

# Start local Spark
docker-compose -f docker-compose.spark.yaml up -d

# Run job
python -m src.ods_sync.main --env local

# Or with specific tables
python -m src.ods_sync.main --env local --tables orders,quotes
```

## Run on EMR

```bash
# Upload job to S3
aws s3 cp src/ods_sync/ s3://maip-${MAIP_ENV}-artifacts/spark/ods_sync/ --recursive
aws s3 cp requirements.txt s3://maip-${MAIP_ENV}-artifacts/spark/

# Submit job
aws emr add-steps --cluster-id $EMR_CLUSTER_ID --steps '[{
  "Name": "MAIP ODS Sync - '$(date +%Y%m%d-%H%M)'",
  "Type": "Spark",
  "ActionOnFailure": "CONTINUE",
  "Args": [
    "--deploy-mode", "cluster",
    "--py-files", "s3://maip-'${MAIP_ENV}'-artifacts/spark/dependencies.zip",
    "s3://maip-'${MAIP_ENV}'-artifacts/spark/ods_sync/main.py",
    "--env", "'${MAIP_ENV}'"
  ]
}]'
```

## Check Job Status

```bash
# List recent steps
aws emr list-steps --cluster-id $EMR_CLUSTER_ID --max-items 5

# Get step details
aws emr describe-step --cluster-id $EMR_CLUSTER_ID --step-id {step-id}
```

## Code Structure

```python
# main.py
from pyspark.sql import SparkSession
from opentelemetry import trace

tracer = trace.get_tracer("maip.spark.ods-sync")

def sync_table(spark, source_table, target_table):
    with tracer.start_as_current_span(f"sync_{source_table}") as span:
        df = spark.read.format("dynamodb").option("tableName", source_table).load()
        span.set_attribute("record.count", df.count())
        
        # Transform and write to analytics table
        transformed = transform_for_analytics(df)
        transformed.write.format("dynamodb").option("tableName", target_table).mode("append").save()

def main(env):
    spark = SparkSession.builder.appName("MAIP-ODS-Sync").getOrCreate()
    
    tables = ["products", "customers", "applications", "quotes", "orders"]
    for table in tables:
        sync_table(spark, f"maip-{env}-{table}", f"maip-{env}-merchant-analytics")
```

## Schedule (Production)
Runs every 15 minutes via AWS Step Functions.

## Monitoring

```bash
# Check Spark UI (via SSH tunnel to EMR master)
ssh -L 18080:localhost:18080 hadoop@$MASTER_DNS
# Open http://localhost:18080
```
