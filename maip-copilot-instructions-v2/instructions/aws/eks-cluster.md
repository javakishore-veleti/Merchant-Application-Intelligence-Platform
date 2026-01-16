# EKS Cluster Management

## Purpose

Manage MAIP Kubernetes clusters on Amazon EKS. All microservices run in the `maip-services` namespace.

---

## Prerequisites

```bash
# ALWAYS set profile first
export AWS_PROFILE=maip-${MAIP_ENV:-dev} AWS_REGION=us-east-1

# Verify kubectl installed
kubectl version --client
```

---

## Cluster Names

| Environment | Cluster Name |
|-------------|--------------|
| dev | `maip-dev-eks-cluster` |
| qa | `maip-qa-eks-cluster` |
| staging | `maip-staging-eks-cluster` |
| prod | `maip-prod-eks-cluster` |

---

## Validated Commands

### Configure kubectl (Run Once Per Session)

```bash
aws eks update-kubeconfig --name maip-${MAIP_ENV:-dev}-eks-cluster
```

**Expected Output:**
```
Added new context arn:aws:eks:us-east-1:123456789012:cluster/maip-dev-eks-cluster to /home/user/.kube/config
```

### View Resources

```bash
# List nodes
kubectl get nodes

# List all pods in MAIP namespace
kubectl get pods -n maip-services

# List services
kubectl get svc -n maip-services

# List deployments
kubectl get deployments -n maip-services

# List HPA
kubectl get hpa -n maip-services
```

### View Logs

```bash
# Tail logs for a service (most common)
kubectl logs -f -l app={service-name} -n maip-services --tail=100

# Logs from specific pod
kubectl logs {pod-name} -n maip-services

# Previous container logs (after crash)
kubectl logs {pod-name} -n maip-services --previous
```

### Scale Deployment

```bash
# Scale to specific replica count
kubectl scale deployment {service-name} -n maip-services --replicas={count}

# Example: Scale order-service to 8 replicas
kubectl scale deployment order-service -n maip-services --replicas=8
```

**Validation:** `kubectl get pods -n maip-services -l app={service-name}`

### Restart Deployment

```bash
# Restart (rolling)
kubectl rollout restart deployment/{service-name} -n maip-services

# Check status
kubectl rollout status deployment/{service-name} -n maip-services
```

**Expected Output:**
```
deployment "order-service" successfully rolled out
```

### Rollback

```bash
# View history
kubectl rollout history deployment/{service-name} -n maip-services

# Rollback to previous
kubectl rollout undo deployment/{service-name} -n maip-services

# Rollback to specific revision
kubectl rollout undo deployment/{service-name} -n maip-services --to-revision={revision}
```

### Debug Pod Issues

```bash
# Describe pod (shows events, errors)
kubectl describe pod -l app={service-name} -n maip-services

# Exec into pod
kubectl exec -it {pod-name} -n maip-services -- /bin/sh

# Check resource usage
kubectl top pods -n maip-services
```

---

## Service Names (Use Exactly)

```
product-catalog-service
customer-profile-service
merchant-application-service
pricing-engine-service
quote-service
order-service
notification-service
audit-service
```

---

## Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `error: context not found` | kubectl not configured | Run `aws eks update-kubeconfig` |
| `Unauthorized` | Wrong AWS profile | Verify `aws sts get-caller-identity` |
| `pods "X" not found` | Wrong namespace | Add `-n maip-services` |
| `ImagePullBackOff` | ECR auth or image missing | Check `kubectl describe pod` |
| `CrashLoopBackOff` | Application error | Check `kubectl logs --previous` |

---

## Node Group Scaling (Admin Only)

```bash
# Scale node group
aws eks update-nodegroup-config \
  --cluster-name maip-${MAIP_ENV}-eks-cluster \
  --nodegroup-name maip-${MAIP_ENV}-nodegroup \
  --scaling-config minSize=2,maxSize=20,desiredSize={count}
```

**Time:** ~5 minutes for nodes to join
