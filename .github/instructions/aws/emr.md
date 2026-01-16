# EMR Management - MAIP

## Cluster Names
- `maip-staging-emr`
- `maip-prod-emr`

(Dev/QA use local Docker Spark/Flink)

## Get Cluster ID

```bash
export EMR_CLUSTER_ID=$(aws emr list-clusters --active \
  --query "Clusters[?Name=='maip-${MAIP_ENV}-emr'].Id" --output text)

echo $EMR_CLUSTER_ID
```

## Check Cluster Status

```bash
aws emr describe-cluster --cluster-id $EMR_CLUSTER_ID \
  --query 'Cluster.{Status:Status.State,Name:Name}'
```

## Submit Spark Job - ODS Sync

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

## Submit Spark Job - Analytics

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

## Submit Flink Job - Stream Processor

```bash
aws emr add-steps --cluster-id $EMR_CLUSTER_ID --steps '[{
  "Name": "MAIP Flink Stream Processor",
  "Type": "Custom",
  "ActionOnFailure": "CONTINUE",
  "Jar": "command-runner.jar",
  "Args": [
    "flink", "run", "-d",
    "s3://maip-'${MAIP_ENV}'-artifacts/flink/maip-stream-processor.jar",
    "--kafka.bootstrap", "'$MAIP_KAFKA_BROKERS'",
    "--env", "'${MAIP_ENV}'"
  ]
}]'
```

## Check Step Status

```bash
# List recent steps
aws emr list-steps --cluster-id $EMR_CLUSTER_ID --max-items 5

# Describe specific step
aws emr describe-step --cluster-id $EMR_CLUSTER_ID --step-id {step-id}
```

## Cancel Step

```bash
aws emr cancel-steps --cluster-id $EMR_CLUSTER_ID --step-ids {step-id}
```

## Scale Cluster

```bash
# Get instance group ID
CORE_GROUP_ID=$(aws emr list-instance-groups --cluster-id $EMR_CLUSTER_ID \
  --query "InstanceGroups[?InstanceGroupType=='CORE'].Id" --output text)

# Scale core nodes
aws emr modify-instance-groups --cluster-id $EMR_CLUSTER_ID \
  --instance-groups InstanceGroupId=$CORE_GROUP_ID,InstanceCount={count}
```

## SSH to Master Node

```bash
MASTER_DNS=$(aws emr describe-cluster --cluster-id $EMR_CLUSTER_ID \
  --query 'Cluster.MasterPublicDnsName' --output text)

ssh -i ~/.ssh/maip-emr-key.pem hadoop@$MASTER_DNS
```

## View Spark UI (via SSH Tunnel)

```bash
ssh -i ~/.ssh/maip-emr-key.pem -L 18080:localhost:18080 hadoop@$MASTER_DNS
# Then open http://localhost:18080
```

## Upload Artifacts to S3

```bash
# Upload Spark job
aws s3 cp backend/data-services/spark-jobs/dist/ods_sync.py \
  s3://maip-${MAIP_ENV}-artifacts/spark/

# Upload Flink jar
aws s3 cp backend/data-services/flink-jobs/target/maip-stream-processor.jar \
  s3://maip-${MAIP_ENV}-artifacts/flink/
```
