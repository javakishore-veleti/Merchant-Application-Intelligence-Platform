# EMR Spark/Flink Management

## Purpose

Manage Amazon EMR clusters for MAIP batch (Spark) and stream (Flink) processing jobs.

---

## Prerequisites

```bash
# ALWAYS set profile first
export AWS_PROFILE=maip-${MAIP_ENV:-dev} AWS_REGION=us-east-1

# Get EMR cluster ID
export EMR_CLUSTER_ID=$(aws emr list-clusters --active \
  --query "Clusters[?Name=='maip-${MAIP_ENV:-staging}-emr'].Id" --output text)

echo $EMR_CLUSTER_ID  # Verify not empty
```

**Note:** Dev/QA use local Docker. EMR is for staging/prod only.

---

## Cluster Names

| Environment | Cluster Name |
|-------------|--------------|
| staging | `maip-staging-emr` |
| prod | `maip-prod-emr` |

---

## Validated Commands

### Check Cluster Status

```bash
aws emr describe-cluster --cluster-id $EMR_CLUSTER_ID \
  --query 'Cluster.{Name:Name,Status:Status.State}'
```

**Expected Output:**
```json
{
    "Name": "maip-staging-emr",
    "Status": "WAITING"
}
```

### Submit Spark Job (ODS Sync)

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

**Expected Output:**
```json
{
    "StepIds": ["s-XXXXXXXXXXXXX"]
}
```

### Submit Spark Job (Analytics)

```bash
aws emr add-steps --cluster-id $EMR_CLUSTER_ID --steps '[{
  "Name": "MAIP Analytics - '$(date +%Y%m%d-%H%M)'",
  "Type": "Spark",
  "ActionOnFailure": "CONTINUE",
  "Args": [
    "--deploy-mode", "cluster",
    "s3://maip-'${MAIP_ENV}'-artifacts/spark/merchant_analytics.py",
    "--env", "'${MAIP_ENV}'"
  ]
}]'
```

### Submit Flink Job

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

### Check Step Status

```bash
# List recent steps
aws emr list-steps --cluster-id $EMR_CLUSTER_ID --max-items 5

# Get specific step status
aws emr describe-step --cluster-id $EMR_CLUSTER_ID --step-id {step-id}
```

**Status Values:** `PENDING`, `RUNNING`, `COMPLETED`, `CANCELLED`, `FAILED`

### Cancel Step

```bash
aws emr cancel-steps --cluster-id $EMR_CLUSTER_ID --step-ids {step-id}
```

### Scale Cluster

```bash
# Get instance group ID
CORE_GROUP_ID=$(aws emr list-instance-groups --cluster-id $EMR_CLUSTER_ID \
  --query "InstanceGroups[?InstanceGroupType=='CORE'].Id" --output text)

# Scale core nodes
aws emr modify-instance-groups --cluster-id $EMR_CLUSTER_ID \
  --instance-groups InstanceGroupId=$CORE_GROUP_ID,InstanceCount={count}
```

**Time:** ~5-10 minutes for new nodes

---

## Upload Artifacts to S3

```bash
# Upload Spark job
aws s3 cp backend/data-services/spark-jobs/src/ods_sync/main.py \
  s3://maip-${MAIP_ENV}-artifacts/spark/ods_sync.py

# Upload dependencies
cd backend/data-services/spark-jobs
pip install -r requirements.txt -t ./package
cd package && zip -r ../dependencies.zip .
aws s3 cp ../dependencies.zip s3://maip-${MAIP_ENV}-artifacts/spark/

# Upload Flink JAR
aws s3 cp backend/data-services/flink-jobs/target/maip-stream-processor.jar \
  s3://maip-${MAIP_ENV}-artifacts/flink/
```

---

## SSH to Master Node

```bash
# Get master DNS
MASTER_DNS=$(aws emr describe-cluster --cluster-id $EMR_CLUSTER_ID \
  --query 'Cluster.MasterPublicDnsName' --output text)

# SSH
ssh -i ~/.ssh/maip-emr-key.pem hadoop@$MASTER_DNS
```

### Access Spark UI (via SSH Tunnel)

```bash
ssh -i ~/.ssh/maip-emr-key.pem -L 18080:localhost:18080 hadoop@$MASTER_DNS
# Open http://localhost:18080
```

### Access Flink UI (via SSH Tunnel)

```bash
ssh -i ~/.ssh/maip-emr-key.pem -L 8081:localhost:8081 hadoop@$MASTER_DNS
# Open http://localhost:8081
```

---

## Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `Cluster not found` | Wrong ID or terminated | Re-run `export EMR_CLUSTER_ID=...` |
| `Step failed` | Job error | Check `aws emr describe-step` for logs |
| `S3 access denied` | IAM policy | Verify EMR role has S3 access |
| `Out of memory` | Executor OOM | Increase `--driver-memory` or `--executor-memory` |

---

## Emergency: Job Stuck

```bash
# 1. Check step status
aws emr describe-step --cluster-id $EMR_CLUSTER_ID --step-id {step-id}

# 2. Cancel if needed
aws emr cancel-steps --cluster-id $EMR_CLUSTER_ID --step-ids {step-id}

# 3. Check logs in S3
aws s3 ls s3://maip-${MAIP_ENV}-logs/emr/{cluster-id}/steps/{step-id}/
aws s3 cp s3://maip-${MAIP_ENV}-logs/emr/{cluster-id}/steps/{step-id}/stderr.gz - | gunzip
```
