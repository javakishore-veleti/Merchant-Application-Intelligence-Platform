# Spark ODS Sync Jobs

## Purpose

PySpark batch jobs that consolidate data from domain-specific databases into the unified `merchant-analytics` DynamoDB table.

---

## Location

`backend/data-services/spark-jobs/`

---

## Prerequisites

```bash
# Python 3.11+
python --version

# Install dependencies
cd backend/data-services/spark-jobs
pip install -r requirements.txt
```

---

## Source Tables

- `maip-{env}-products`
- `maip-{env}-customers`
- `maip-{env}-applications`
- `maip-{env}-quotes`
- `maip-{env}-orders`

## Target Table

- `maip-{env}-merchant-analytics`

---

## Run Locally

### Start Local Spark

```bash
cd backend/data-services/spark-jobs
docker-compose -f docker-compose.spark.yaml up -d
```

### Run ODS Sync

```bash
python -m src.ods_sync.main --env local
```

### Run with Specific Tables

```bash
python -m src.ods_sync.main --env local --tables orders,quotes
```

---

## Run on EMR

### Upload Artifacts

```bash
# Upload main job
aws s3 cp src/ods_sync/main.py \
  s3://maip-${MAIP_ENV}-artifacts/spark/ods_sync.py

# Create and upload dependencies
pip install -r requirements.txt -t ./package
cd package && zip -r ../dependencies.zip . && cd ..
aws s3 cp dependencies.zip s3://maip-${MAIP_ENV}-artifacts/spark/
```

### Submit Job

```bash
aws emr add-steps --cluster-id $EMR_CLUSTER_ID --steps '[{
  "Name": "MAIP ODS Sync - '$(date +%Y%m%d-%H%M)'",
  "Type": "Spark",
  "ActionOnFailure": "CONTINUE",
  "Args": [
    "--deploy-mode", "cluster",
    "--py-files", "s3://maip-'${MAIP_ENV}'-artifacts/spark/dependencies.zip",
    "s3://maip-'${MAIP_ENV}'-artifacts/spark/ods_sync.py",
    "--env", "'${MAIP_ENV}'"
  ]
}]'
```

### Check Status

```bash
aws emr list-steps --cluster-id $EMR_CLUSTER_ID --max-items 5
```

---

## Code Structure

```python
# main.py
from pyspark.sql import SparkSession
from opentelemetry import trace

tracer = trace.get_tracer("maip.spark.ods-sync")

def sync_table(spark, source_table, target_table):
    with tracer.start_as_current_span(f"sync_{source_table}") as span:
        df = spark.read.format("dynamodb") \
            .option("tableName", source_table).load()
        span.set_attribute("record.count", df.count())
        
        transformed = transform_for_analytics(df)
        transformed.write.format("dynamodb") \
            .option("tableName", target_table) \
            .mode("append").save()

def main(env):
    spark = SparkSession.builder \
        .appName("MAIP-ODS-Sync").getOrCreate()
    
    tables = ["products", "customers", "applications", "quotes", "orders"]
    for table in tables:
        sync_table(spark, f"maip-{env}-{table}", 
                   f"maip-{env}-merchant-analytics")
```

---

## Test Commands

```bash
cd backend/data-services/spark-jobs

# Run all tests
pytest

# Run specific test
pytest tests/test_ods_sync.py -v

# Run with coverage
pytest --cov=src tests/
```

---

## Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `Table not found` | Wrong env or table name | Verify DynamoDB table exists |
| `OutOfMemoryError` | Data too large | Increase `--driver-memory` |
| `Access denied` | IAM permissions | Check EMR role has DynamoDB access |
| `Module not found` | Dependencies missing | Re-upload dependencies.zip |

---

## Scheduling

Production runs every 15 minutes via AWS Step Functions.

Manual trigger:
```bash
aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:us-east-1:${AWS_ACCOUNT}:stateMachine:maip-ods-sync
```
