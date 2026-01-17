# Be A Modern AWS Developer: Manage AWS Infrastructure and Microservices with GitHub Copilot Instructions

---

## Objective

AWS infrastructure management for a Merchant Banking Customer Onboarding Platform becomes effortless with GitHub Copilot Instructions in IntelliJ IDEA, VS Code, or PyCharm. This newsletter shows how instruction files transform your IDE into a command center for AWS CLI, EKS, MSK Kafka, and microservice deployments—using local AWS profiles.

---

GitHub Copilot is remarkably good at generating generic code. Ask it to write a Kafka consumer, and you'll get a textbook implementation. But ask it to scale your specific Kafka consumer group during a transaction surge? It has no idea what your cluster is named, what AWS profile to use, or that your team prefixes everything with `maip-`.

Until you teach it.

---

## The Aha Moment

One of our engineers spent 20 minutes last week trying to figure out the correct AWS CLI command to check MSK consumer lag. The documentation was scattered across Confluence, Slack threads, and someone's personal notes.

Then she tried something different. She added this to our `instructions/aws/msk-kafka.md`:

```markdown
## Check Consumer Lag - MAIP Production

# Step 1: Set AWS profile and environment
export AWS_PROFILE=maip-prod AWS_REGION=us-east-1 MAIP_ENV=prod

# Step 2: Get Kafka broker endpoints
export MAIP_KAFKA_BROKERS=$(aws kafka get-bootstrap-brokers \
  --cluster-arn $(aws kafka list-clusters \
    --query "ClusterInfoList[?ClusterName=='maip-prod-kafka'].ClusterArn" \
    --output text) \
  --query 'BootstrapBrokerStringSasl' --output text)

# Step 3: Check consumer lag
kafka-consumer-groups.sh --bootstrap-server $MAIP_KAFKA_BROKERS \
  --describe --group maip-order-service-consumer
```

Next time someone on the team types "check order consumer lag in production" in their IDE, Copilot generates the exact commands. Correct profile. Correct cluster. Correct consumer group name.

20 minutes became 10 seconds.

---

## What Makes Instructions Actually Useful

**Generic (Copilot ignores this):**
```markdown
Use kubectl to manage Kubernetes resources.
```

**Specific (Copilot uses this):**
```markdown
# Scale Order Service During High Load

# First, ensure you're connected to the right cluster
export AWS_PROFILE=maip-prod AWS_REGION=us-east-1
aws eks update-kubeconfig --name maip-prod-eks-cluster

# Scale the deployment
kubectl scale deployment order-service -n maip-services --replicas=12

# Verify scaling
kubectl get pods -n maip-services -l app=order-service

# Check HPA status
kubectl get hpa -n maip-services
```

The difference: actual deployment names, actual namespaces, actual label selectors. Copilot pattern-matches against these when generating suggestions.

---

## Three Files That Changed Our Operations

We have dozens of instruction files, but three made the biggest immediate impact:

**1. `instructions/aws/eks-cluster.md`**

Contains every EKS command with correct resource names. When the 3 AM alert fires, the on-call engineer types "scale order processing" and gets executable commands, not documentation to read.

**2. `instructions/microservices/pricing-engine-service.md`**

Our pricing engine uses Drools rules. This file contains the fact object names, method signatures, and rule patterns. When adding a new discount rule, Copilot generates syntactically correct Drools—not generic Java.

**3. `instructions/deployment/environments.md`**

QA environment deployment in one shot. The file contains the exact sequence: set profile, configure kubectl, apply manifests, verify. Copilot suggests the whole orchestrated workflow.

---

## The Command That Convinced Our Team

Before instructions:
```bash
# Developer types: "deploy pricing engine to QA"
# Copilot suggests: kubectl apply -f deployment.yaml
# Useless. Which deployment.yaml? Which namespace? Which cluster?
```

After adding deployment instructions:
```bash
# Developer types: "deploy pricing engine to QA"
# Copilot suggests:

# Set QA environment
export AWS_PROFILE=maip-qa AWS_REGION=us-east-1 MAIP_ENV=qa

# Configure kubectl for QA cluster
aws eks update-kubeconfig --name maip-qa-eks-cluster

# Deploy
kubectl apply -f DevOps/Kubernetes/services/pricing-engine/ -n maip-services

# Verify deployment
kubectl rollout status deployment/pricing-engine-service -n maip-services

# Watch logs
kubectl logs -f -l app=pricing-engine-service -n maip-services --tail=50
```

Set environment, connect to cluster, deploy, verify, watch logs. In one suggestion. With correct paths and namespaces.

---

## How It Works in Practice

When you add instruction files to your repo, Copilot reads them as context. Here's the flow:

1. **You type a comment** in your terminal, IDE, or Copilot Chat: `# deploy pricing engine to QA`

2. **Copilot searches your instructions** for relevant patterns (it finds "QA", "pricing-engine", "deploy")

3. **Copilot generates commands** using your specific resource names, not generic placeholders

The key insight: Copilot doesn't execute commands. It *suggests* commands based on your instruction patterns. You review and run them in your actual terminal.

---

## Start Here

Pick the one thing that causes the most friction on your team. For us, it was Kafka operations—checking lag, resetting offsets, creating topics.

Create `instructions/aws/msk-kafka.md` (or whatever matches your setup). Add the five commands your team runs most often, with your actual resource names.

Push it. Wait for the first "how did Copilot know that?" from a teammate.

That's the moment everything changes.

---

**Full repository with all instruction files:**  
[github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/tree/main/instructions](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/tree/main/instructions)