# CloudFormation Deployment

## Purpose

Deploy and manage MAIP AWS infrastructure using CloudFormation templates.

---

## Prerequisites

```bash
# ALWAYS set profile first
export AWS_PROFILE=maip-${MAIP_ENV:-dev} AWS_REGION=us-east-1
```

---

## Stack Naming Convention

Pattern: `maip-{env}-{purpose}`

---

## Deployment Order (New Environment)

**ALWAYS deploy in this order.** Dependencies require it.

| Order | Stack | Template | Time | Dependencies |
|-------|-------|----------|------|--------------|
| 1 | `maip-{env}-networking` | `networking/vpc.yaml` | ~5 min | None |
| 2 | `maip-{env}-security` | `security/security-groups.yaml` | ~2 min | networking |
| 3 | `maip-{env}-rds` | `database/rds-postgres.yaml` | ~15 min | networking, security |
| 4 | `maip-{env}-dynamodb` | `database/dynamodb-tables.yaml` | ~5 min | None |
| 5 | `maip-{env}-msk` | `streaming/msk-cluster.yaml` | ~25 min | networking, security |
| 6 | `maip-{env}-eks` | `compute/eks-cluster.yaml` | ~20 min | networking, security |
| 7 | `maip-{env}-observability` | `observability/opensearch.yaml` | ~15 min | networking, security |

---

## Validated Commands

### Deploy Stack

```bash
aws cloudformation deploy \
  --template-file DevOps/Cloud/AWS/cloudformation/{category}/{template}.yaml \
  --stack-name maip-${MAIP_ENV}-{purpose} \
  --parameter-overrides file://DevOps/Cloud/AWS/cloudformation/parameters/${MAIP_ENV}/{purpose}.json \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
  --tags Project=MAIP Environment=${MAIP_ENV}
```

### Validate Template (Before Deploy)

```bash
aws cloudformation validate-template \
  --template-body file://DevOps/Cloud/AWS/cloudformation/{category}/{template}.yaml
```

### Check Stack Status

```bash
aws cloudformation describe-stacks \
  --stack-name maip-${MAIP_ENV}-{purpose} \
  --query 'Stacks[0].StackStatus'
```

**Expected Values:** `CREATE_COMPLETE`, `UPDATE_COMPLETE`

### List All MAIP Stacks

```bash
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --query "StackSummaries[?contains(StackName, 'maip-${MAIP_ENV}')].{Name:StackName,Status:StackStatus}"
```

### Get Stack Outputs

```bash
aws cloudformation describe-stacks \
  --stack-name maip-${MAIP_ENV}-networking \
  --query 'Stacks[0].Outputs'
```

### View Stack Events (Debugging)

```bash
aws cloudformation describe-stack-events \
  --stack-name maip-${MAIP_ENV}-{purpose} \
  --query 'StackEvents[0:10].{Time:Timestamp,Resource:LogicalResourceId,Status:ResourceStatus,Reason:ResourceStatusReason}'
```

### Update Stack

```bash
aws cloudformation deploy \
  --template-file DevOps/Cloud/AWS/cloudformation/{category}/{template}.yaml \
  --stack-name maip-${MAIP_ENV}-{purpose} \
  --parameter-overrides file://parameters/${MAIP_ENV}/{purpose}.json \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
  --no-fail-on-empty-changeset
```

### Delete Stack (Careful!)

```bash
# Delete single stack
aws cloudformation delete-stack --stack-name maip-${MAIP_ENV}-{purpose}

# Wait for deletion
aws cloudformation wait stack-delete-complete --stack-name maip-${MAIP_ENV}-{purpose}
```

**Warning:** Delete in reverse order of deployment.

---

## Full Environment Deployment Script

```bash
#!/bin/bash
set -e
ENV=${1:-dev}
export AWS_PROFILE=maip-$ENV AWS_REGION=us-east-1

echo "Deploying MAIP $ENV environment..."

# 1. Networking
echo "1/6 Deploying networking..."
aws cloudformation deploy \
  --template-file DevOps/Cloud/AWS/cloudformation/networking/vpc.yaml \
  --stack-name maip-$ENV-networking \
  --parameter-overrides file://parameters/$ENV/networking.json \
  --capabilities CAPABILITY_IAM

# 2. Security
echo "2/6 Deploying security..."
aws cloudformation deploy \
  --template-file DevOps/Cloud/AWS/cloudformation/security/security-groups.yaml \
  --stack-name maip-$ENV-security \
  --parameter-overrides file://parameters/$ENV/security.json

# 3. RDS
echo "3/6 Deploying RDS..."
aws cloudformation deploy \
  --template-file DevOps/Cloud/AWS/cloudformation/database/rds-postgres.yaml \
  --stack-name maip-$ENV-rds \
  --parameter-overrides file://parameters/$ENV/rds.json \
  --capabilities CAPABILITY_IAM

# 4. DynamoDB
echo "4/6 Deploying DynamoDB..."
aws cloudformation deploy \
  --template-file DevOps/Cloud/AWS/cloudformation/database/dynamodb-tables.yaml \
  --stack-name maip-$ENV-dynamodb \
  --parameter-overrides file://parameters/$ENV/dynamodb.json

# 5. MSK
echo "5/6 Deploying MSK..."
aws cloudformation deploy \
  --template-file DevOps/Cloud/AWS/cloudformation/streaming/msk-cluster.yaml \
  --stack-name maip-$ENV-msk \
  --parameter-overrides file://parameters/$ENV/msk.json

# 6. EKS
echo "6/6 Deploying EKS..."
aws cloudformation deploy \
  --template-file DevOps/Cloud/AWS/cloudformation/compute/eks-cluster.yaml \
  --stack-name maip-$ENV-eks \
  --parameter-overrides file://parameters/$ENV/eks.json \
  --capabilities CAPABILITY_NAMED_IAM

echo "MAIP $ENV infrastructure deployed!"
```

---

## Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `ROLLBACK_COMPLETE` | Creation failed | Check events: `describe-stack-events` |
| `No updates to perform` | No changes | Add `--no-fail-on-empty-changeset` |
| `Insufficient IAM permissions` | Missing capabilities | Add `--capabilities CAPABILITY_IAM` |
| `Resource already exists` | Naming conflict | Use unique stack names |
| `DELETE_FAILED` | Resources in use | Manually delete dependent resources |

---

## Emergency: Stack Stuck

```bash
# 1. Check events
aws cloudformation describe-stack-events \
  --stack-name maip-${MAIP_ENV}-{purpose} --max-items 20

# 2. If stuck in DELETE_IN_PROGRESS, force delete
aws cloudformation delete-stack \
  --stack-name maip-${MAIP_ENV}-{purpose} \
  --retain-resources {resource-logical-id}
```
