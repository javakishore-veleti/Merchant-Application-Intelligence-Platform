# CloudFormation Deployment - MAIP

## Stack Naming
Pattern: `maip-{env}-{stack-purpose}`

## Stacks and Deploy Order

| Order | Stack | Template | Time |
|-------|-------|----------|------|
| 1 | maip-{env}-networking | networking/vpc.yaml | ~5 min |
| 2 | maip-{env}-security | security/security-groups.yaml | ~2 min |
| 3 | maip-{env}-rds | database/rds-postgres.yaml | ~15 min |
| 4 | maip-{env}-dynamodb | database/dynamodb-tables.yaml | ~5 min |
| 5 | maip-{env}-msk | streaming/msk-cluster.yaml | ~25 min |
| 6 | maip-{env}-eks | compute/eks-cluster.yaml | ~20 min |
| 7 | maip-{env}-observability | observability/opensearch.yaml | ~15 min |
| 8 | maip-{env}-bedrock | ai/bedrock-integration.yaml | ~3 min |

## Deploy Single Stack

```bash
maip-dev && aws cloudformation deploy \
  --template-file DevOps/Cloud/AWS/cloudformation/{category}/{template}.yaml \
  --stack-name maip-${MAIP_ENV}-{purpose} \
  --parameter-overrides file://DevOps/Cloud/AWS/cloudformation/parameters/${MAIP_ENV}/{purpose}.json \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
  --tags Project=MAIP Environment=${MAIP_ENV}
```

## Deploy All (New Environment)

```bash
#!/bin/bash
set -e

ENV=${1:-dev}
maip-$ENV

echo "Deploying MAIP $ENV environment..."

# 1. Networking
aws cloudformation deploy \
  --template-file DevOps/Cloud/AWS/cloudformation/networking/vpc.yaml \
  --stack-name maip-$ENV-networking \
  --parameter-overrides file://parameters/$ENV/networking.json \
  --capabilities CAPABILITY_IAM

# 2. Security Groups
aws cloudformation deploy \
  --template-file DevOps/Cloud/AWS/cloudformation/security/security-groups.yaml \
  --stack-name maip-$ENV-security \
  --parameter-overrides file://parameters/$ENV/security.json

# 3. RDS
aws cloudformation deploy \
  --template-file DevOps/Cloud/AWS/cloudformation/database/rds-postgres.yaml \
  --stack-name maip-$ENV-rds \
  --parameter-overrides file://parameters/$ENV/rds.json \
  --capabilities CAPABILITY_IAM

# 4. DynamoDB
aws cloudformation deploy \
  --template-file DevOps/Cloud/AWS/cloudformation/database/dynamodb-tables.yaml \
  --stack-name maip-$ENV-dynamodb \
  --parameter-overrides file://parameters/$ENV/dynamodb.json

# 5. MSK
aws cloudformation deploy \
  --template-file DevOps/Cloud/AWS/cloudformation/streaming/msk-cluster.yaml \
  --stack-name maip-$ENV-msk \
  --parameter-overrides file://parameters/$ENV/msk.json

# 6. EKS
aws cloudformation deploy \
  --template-file DevOps/Cloud/AWS/cloudformation/compute/eks-cluster.yaml \
  --stack-name maip-$ENV-eks \
  --parameter-overrides file://parameters/$ENV/eks.json \
  --capabilities CAPABILITY_NAMED_IAM

echo "MAIP $ENV infrastructure deployed!"
```

## Check Stack Status

```bash
aws cloudformation describe-stacks \
  --stack-name maip-${MAIP_ENV}-{purpose} \
  --query 'Stacks[0].StackStatus'
```

## List All MAIP Stacks

```bash
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --query "StackSummaries[?contains(StackName, 'maip-${MAIP_ENV}')].{Name:StackName,Status:StackStatus}"
```

## Get Stack Outputs

```bash
aws cloudformation describe-stacks \
  --stack-name maip-${MAIP_ENV}-networking \
  --query 'Stacks[0].Outputs'
```

## Update Stack

```bash
aws cloudformation deploy \
  --template-file DevOps/Cloud/AWS/cloudformation/{category}/{template}.yaml \
  --stack-name maip-${MAIP_ENV}-{purpose} \
  --parameter-overrides file://parameters/${MAIP_ENV}/{purpose}.json \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
  --no-fail-on-empty-changeset
```

## Delete Stack (Careful!)

```bash
# Single stack
aws cloudformation delete-stack --stack-name maip-${MAIP_ENV}-{purpose}

# Wait for deletion
aws cloudformation wait stack-delete-complete --stack-name maip-${MAIP_ENV}-{purpose}
```

## Validate Template

```bash
aws cloudformation validate-template \
  --template-body file://DevOps/Cloud/AWS/cloudformation/{category}/{template}.yaml
```

## View Stack Events (Debugging)

```bash
aws cloudformation describe-stack-events \
  --stack-name maip-${MAIP_ENV}-{purpose} \
  --query 'StackEvents[0:10].{Time:Timestamp,Status:ResourceStatus,Reason:ResourceStatusReason}'
```
