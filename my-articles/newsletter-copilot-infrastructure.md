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

Then she tried something different. She added this to our `instructions/aws/msk-kafka-management.md`:

```markdown
## Check Consumer Lag - MAIP Production

maip-prod && kafka-consumer-groups.sh \
  --bootstrap-server $MAIP_PROD_KAFKA \
  --describe --group maip-order-service-consumer

## Environment Variable Setup
export MAIP_PROD_KAFKA=$(aws kafka get-bootstrap-brokers \
  --cluster-arn arn:aws:kafka:us-east-1:ACCOUNT:cluster/maip-prod-kafka/xxx \
  --query 'BootstrapBrokerStringSasl' --output text)
```

Next time someone on the team types "check order consumer lag" in their IDE, Copilot generates the exact command. Correct profile. Correct cluster. Correct consumer group name.

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

kubectl scale deployment order-service \
  -n maip-services --replicas=12

# Verify scaling
kubectl get pods -n maip-services -l app=order-service

# Check HPA status
kubectl get hpa order-service-hpa -n maip-services
```

The difference: actual deployment names, actual namespaces, actual label selectors. Copilot pattern-matches against these when generating suggestions.

---

## Three Files That Changed Our Operations

We have dozens of instruction files, but three made the biggest immediate impact:

**1. `instructions/aws/eks-emergency-scaling.md`**

Contains every emergency scaling command with correct resource names. When the 3 AM alert fires, the on-call engineer types "scale order processing" and gets executable commands, not documentation to read.

**2. `instructions/microservices/pricing-engine-service.md`**

Our pricing engine uses Drools rules. This file contains the fact object names, method signatures, and rule patterns. When adding a new discount rule, Copilot generates syntactically correct Drools—not generic Java.

**3. `instructions/deployment/deploy-qa.md`**

QA environment spinup in one shot. The file contains the exact sequence: networking first, then database, then Kafka topics, then EKS deployments. Copilot suggests the whole orchestrated workflow.

---

## The Command That Convinced Our Team

Before instructions:
```bash
# Developer types: "deploy pricing engine to QA"
# Copilot suggests: kubectl apply -f deployment.yaml
# Useless. Which deployment.yaml? Which namespace? Which context?
```

After adding deployment instructions:
```bash
# Developer types: "deploy pricing engine to QA"
# Copilot suggests:
maip-qa && kubectl apply -f DevOps/Kubernetes/services/pricing-engine/ -n maip-services
kubectl rollout status deployment/pricing-engine-service -n maip-services
kubectl logs -f -l app=pricing-engine-service -n maip-services --tail=50
```

Deploy, verify, watch logs. In one suggestion. With correct paths and namespaces.

---

## Start Here

Pick the one thing that causes the most friction on your team. For us, it was Kafka operations—checking lag, resetting offsets, creating topics.

Create `instructions/aws/kafka-operations.md` (or whatever matches your setup). Add the five commands your team runs most often, with your actual resource names.

Push it. Wait for the first "how did Copilot know that?" from a teammate.

That's the moment everything changes.

---

**Full repository with all instruction files:**  
[github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/instructions/](https://github.com/javakishore-veleti/Merchant-Application-Intelligence-Platform/instructions/)
