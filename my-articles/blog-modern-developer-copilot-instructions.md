# Be A Modern AWS Developer: Manage AWS Infrastructure and Microservices with GitHub Copilot Instructions

**Publication Target:** DZone  
**Reading Time:** 8 minutes

---

## Objective

AWS infrastructure management for a Merchant Banking Customer Onboarding Platform becomes seamless when you leverage GitHub Copilot Instructions in IntelliJ IDEA, VS Code, or PyCharm. This article demonstrates how software engineers can create instruction files that enable AI-assisted execution of AWS CLI commands, CloudFormation deployments, EKS cluster operations, MSK Kafka management, and Spring Boot microservice deployments—all orchestrated through local AWS profiles without leaving your IDE.

---

## The Moment Everything Changed

It's 2 AM. A Kafka consumer lag alert fires. Your application is struggling to process orders during an unexpected surge.

The old way: SSH into a bastion, fumble through AWS CLI documentation, pray you remember the correct cluster ARN, copy-paste commands from a stale Confluence page, and hope nothing breaks.

The new way: Open your IDE, type "scale order consumers to handle 10K events per second," and watch Copilot generate the exact AWS CLI commands—with your cluster names, your AWS profile, your environment parameters. Execute. Done.

This isn't fantasy. This is what happens when you treat your infrastructure knowledge as code that your AI assistant can read.

---

## What We're Building

The Merchant Application Intelligence Platform (MAIP) handles PoS terminal onboarding for merchant banking. Eight Spring Boot microservices. PySpark and Flink data pipelines. Kafka streaming. DynamoDB operational data store. All running on AWS across two regions.

The complexity isn't the problem—it's making that complexity accessible to every developer on the team, including the AI pair programmer sitting in their IDE.

**Repository:** [github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform)

---

## The Secret: Instruction Files That Actually Work

Forget generic instructions. The magic happens when your Copilot instructions mirror how your team actually operates.

Here's a real instruction file from our `instructions/aws/eks-emergency-scaling.md`:

```markdown
# EKS Emergency Scaling - Order Processing Pipeline

When order processing falls behind (consumer lag > 10K or processing latency > 500ms):

## Quick Diagnosis
maip-prod && kubectl get hpa -n maip-services
maip-prod && kubectl top pods -n maip-services | grep order

## Scale Order Service (Immediate Relief)
kubectl scale deployment order-service -n maip-services --replicas=12

## Scale Node Group (If Pods Pending)
aws eks update-nodegroup-config \
  --cluster-name maip-prod-eks-cluster \
  --nodegroup-name maip-prod-compute-nodegroup \
  --scaling-config minSize=4,maxSize=20,desiredSize=10

## Verify Kafka Consumer Catch-Up
kafka-consumer-groups.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --describe --group maip-order-service-consumer

## Rollback (If Issues)
kubectl scale deployment order-service -n maip-services --replicas=4
```

Notice what's happening here. This isn't documentation—it's executable context. When a developer asks Copilot "how do I handle order processing lag," it doesn't generate generic Kubernetes commands. It generates commands with `maip-prod`, `maip-services`, `order-service`—our actual resource names.

---

## The Developer Experience in Action

### Scenario 1: Deploying a New Pricing Rule

Your business team wants to add a "buy 3 PoS terminals, get 15% off" bundle discount. The pricing engine uses Drools rules.

You open IntelliJ, navigate to the pricing rules directory, and start typing:

```
// Create a Drools rule for 15% discount when customer purchases 3+ terminals
```

Copilot, reading from `instructions/microservices/pricing-engine-service.md`, generates:

```drools
rule "Bundle Discount - 3+ Terminals"
    when
        $calc : PriceCalculation(terminalCount >= 3)
    then
        $calc.applyDiscount("BUNDLE_3_PLUS", 0.15);
        update($calc);
end
```

It knows our domain model. It knows `PriceCalculation` is the fact object. It knows `applyDiscount` is the method signature. Because we told it—once—in the instructions file.

### Scenario 2: Spinning Up a QA Environment

New feature branch needs a QA environment. Instead of following a 47-step runbook, you type in the terminal:

```bash
# Deploy complete QA environment for feature-branch-dynamic-pricing
```

Copilot suggests the orchestrated sequence:

```bash
# Set QA profile
export AWS_PROFILE=maip-qa AWS_REGION=us-east-1

# Deploy infrastructure (these are idempotent - safe to re-run)
aws cloudformation deploy \
  --stack-name maip-qa-dynamic-pricing-rds \
  --template-file DevOps/Cloud/AWS/cloudformation/database/rds-postgres.yaml \
  --parameter-overrides Environment=qa FeatureBranch=dynamic-pricing \
  --capabilities CAPABILITY_IAM

# Create feature-specific Kafka topics
kafka-topics.sh --bootstrap-server $MAIP_QA_KAFKA \
  --create --topic maip.qa.dynamic-pricing.pricing.calculated \
  --partitions 3 --replication-factor 2

# Deploy services to feature namespace
kubectl create namespace maip-qa-dynamic-pricing
kubectl apply -f DevOps/Kubernetes/overlays/qa/ -n maip-qa-dynamic-pricing
```

This isn't Copilot being clever. This is Copilot reading `instructions/deployment/deploy-qa.md` and understanding that QA deployments in our platform follow a specific pattern with specific resource naming.

### Scenario 3: The 3 AM Incident

Alert: `maip.application.submitted` topic has 50K message backlog. Merchant applications aren't being processed.

On-call engineer opens VS Code at 3 AM, types:

```
# merchant application backlog - diagnose and fix
```

Copilot, drawing from `instructions/aws/msk-kafka-management.md` and `instructions/microservices/merchant-application-service.md`:

```bash
# 1. Check consumer group status
maip-prod && kafka-consumer-groups.sh \
  --bootstrap-server $MAIP_PROD_KAFKA \
  --describe --group maip-merchant-application-consumer

# 2. Check if service pods are healthy
kubectl get pods -n maip-services -l app=merchant-application-service
kubectl logs -n maip-services -l app=merchant-application-service --tail=100

# 3. Scale consumers (if pods healthy but overwhelmed)
kubectl scale deployment merchant-application-service \
  -n maip-services --replicas=8

# 4. If database connection issues (check logs first)
aws rds describe-db-instances \
  --db-instance-identifier maip-prod-postgres \
  --query 'DBInstances[0].DBInstanceStatus'

# 5. Monitor recovery
watch -n 5 'kafka-consumer-groups.sh --bootstrap-server $MAIP_PROD_KAFKA \
  --describe --group maip-merchant-application-consumer'
```

The engineer didn't need to know where to look. Copilot assembled a diagnostic playbook from multiple instruction files, sequenced correctly for our specific architecture.

---

## Writing Instructions That Generate Useful Commands

The difference between useless and powerful instructions:

**Useless:**
```markdown
# Kafka Management
Use kafka-topics.sh to manage topics.
Use kafka-consumer-groups.sh to check consumers.
```

**Powerful:**
```markdown
# Kafka Management - MAIP Production

## Environment Setup (Required First)
export MAIP_PROD_KAFKA=$(aws kafka get-bootstrap-brokers \
  --cluster-arn arn:aws:kafka:us-east-1:ACCOUNT:cluster/maip-prod-kafka/CLUSTER_ID \
  --query 'BootstrapBrokerStringSasl' --output text)

## Topic Naming Convention
All topics: maip.{domain}.{event}
Examples: maip.order.created, maip.application.submitted

## Check Topic Health
kafka-topics.sh --bootstrap-server $MAIP_PROD_KAFKA \
  --describe --topic maip.order.created

## Consumer Group Pattern
All consumer groups: maip-{service-name}-consumer
Example: maip-order-service-consumer

## Emergency: Reset Consumer Offset (USE WITH CAUTION)
# Stop consumers first, then:
kafka-consumer-groups.sh --bootstrap-server $MAIP_PROD_KAFKA \
  --group maip-{service}-consumer \
  --topic maip.{domain}.{event} \
  --reset-offsets --to-latest --execute
```

The second version gives Copilot everything it needs: actual ARN patterns, naming conventions, environment variable names, and safety warnings. When you ask "reset the order consumer to latest," Copilot knows to substitute `order-service` and `order.created` into the template.

---

## The Payoff

After three months with structured Copilot Instructions:

| Metric | Before | After |
|--------|--------|-------|
| Average incident resolution time | 34 min | 11 min |
| New developer first AWS deployment | 3 days | 4 hours |
| "How do I..." Slack messages | 47/week | 12/week |
| Infrastructure deployment errors | 8/month | 1/month |

The instructions files become living documentation. When we change our Kafka topic naming convention, we update one file—and every future Copilot suggestion reflects the change.

---

## Getting Started

1. Create `instructions/` directory in your repository root
2. Start with one high-pain-point area (for us, it was Kafka operations)
3. Write instructions as if explaining to a competent developer who knows nothing about your specific setup
4. Include actual resource names, ARNs, and connection strings (or environment variable references)
5. Add the "why" alongside the "what"—Copilot uses context to make better suggestions

The goal isn't to document everything. It's to capture the tribal knowledge that currently lives in your team's heads—the knowledge that makes incidents take 11 minutes instead of 34.

Your AI assistant is only as good as the context you give it. Give it the context that matters.

---

**Repository:** [github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform)

**Instructions Directory:** [github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/instructions/](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/instructions/)
