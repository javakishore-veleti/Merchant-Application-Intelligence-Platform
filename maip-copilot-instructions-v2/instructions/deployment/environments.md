# Environment Deployment

## Purpose

Deploy MAIP to AWS environments (dev, qa, staging, prod).

---

## Environments

| Environment | AWS Profile | Region(s) | Purpose |
|-------------|-------------|-----------|---------|
| dev | `maip-dev` | us-east-1 | Development |
| qa | `maip-qa` | us-east-1 | Testing |
| staging | `maip-staging` | us-east-1, us-west-2 | Pre-production |
| prod | `maip-prod` | us-east-1, us-west-2 | Production |

---

## Prerequisites

```bash
# Set profile (ALWAYS first)
export AWS_PROFILE=maip-${MAIP_ENV} AWS_REGION=us-east-1

# Verify
aws sts get-caller-identity
```

---

## Deploy to Dev

```bash
export AWS_PROFILE=maip-dev AWS_REGION=us-east-1 MAIP_ENV=dev

# Configure kubectl
aws eks update-kubeconfig --name maip-dev-eks-cluster

# Deploy services
kubectl apply -f DevOps/Kubernetes/namespaces/maip-namespace.yaml
kubectl apply -f DevOps/Kubernetes/services/ -n maip-services

# Verify
kubectl get pods -n maip-services
```

---

## Deploy to QA

```bash
export AWS_PROFILE=maip-qa AWS_REGION=us-east-1 MAIP_ENV=qa

# Configure kubectl
aws eks update-kubeconfig --name maip-qa-eks-cluster

# Deploy
kubectl apply -f DevOps/Kubernetes/services/ -n maip-services

# Run smoke tests
./scripts/smoke-test.sh qa
```

---

## Deploy to Staging

```bash
export AWS_PROFILE=maip-staging AWS_REGION=us-east-1 MAIP_ENV=staging

# Primary region
aws eks update-kubeconfig --name maip-staging-eks-cluster
kubectl apply -f DevOps/Kubernetes/services/ -n maip-services

# DR region
export AWS_REGION=us-west-2
aws eks update-kubeconfig --name maip-staging-eks-cluster --region us-west-2
kubectl apply -f DevOps/Kubernetes/services/ -n maip-services
```

---

## Deploy to Production

**IMPORTANT:** Use GitHub Actions for production deployments. Manual steps for emergency only.

```bash
export AWS_PROFILE=maip-prod AWS_REGION=us-east-1 MAIP_ENV=prod

# Primary region
aws eks update-kubeconfig --name maip-prod-eks-cluster
kubectl apply -f DevOps/Kubernetes/services/ -n maip-services

# Verify
kubectl get pods -n maip-services
./scripts/smoke-test.sh prod

# DR region
export AWS_REGION=us-west-2
aws eks update-kubeconfig --name maip-prod-eks-cluster --region us-west-2
kubectl apply -f DevOps/Kubernetes/services/ -n maip-services
```

---

## Deploy Single Service

### Build and Push Image

```bash
cd backend/microservices/{service}
VERSION=$(git rev-parse --short HEAD)

# Build
docker build -t maip/{service}:${VERSION} .

# Tag for ECR
docker tag maip/{service}:${VERSION} \
  ${AWS_ACCOUNT}.dkr.ecr.us-east-1.amazonaws.com/maip/{service}:${VERSION}

# Login to ECR
aws ecr get-login-password | docker login --username AWS \
  --password-stdin ${AWS_ACCOUNT}.dkr.ecr.us-east-1.amazonaws.com

# Push
docker push ${AWS_ACCOUNT}.dkr.ecr.us-east-1.amazonaws.com/maip/{service}:${VERSION}
```

### Update Deployment

```bash
kubectl set image deployment/{service} \
  {service}=${AWS_ACCOUNT}.dkr.ecr.us-east-1.amazonaws.com/maip/{service}:${VERSION} \
  -n maip-services

# Verify
kubectl rollout status deployment/{service} -n maip-services
```

---

## Rollback

```bash
# View history
kubectl rollout history deployment/{service} -n maip-services

# Rollback to previous
kubectl rollout undo deployment/{service} -n maip-services

# Rollback to specific revision
kubectl rollout undo deployment/{service} -n maip-services --to-revision={revision}
```

---

## Health Checks

```bash
# All pods healthy
kubectl get pods -n maip-services

# Service endpoints
kubectl get svc -n maip-services

# Test specific service
curl https://api.maip.example.com/api/v1/{service}/actuator/health
```

---

## Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `ImagePullBackOff` | ECR auth or missing image | Re-login to ECR, verify image pushed |
| `CrashLoopBackOff` | Application error | Check logs: `kubectl logs -l app={service}` |
| `Pending` | Insufficient resources | Check node capacity, scale node group |
| `Service unavailable` | Not enough replicas | Scale deployment |

---

## Deployment Checklist

- [ ] Set correct AWS profile
- [ ] Configure kubectl for cluster
- [ ] Build and push Docker image
- [ ] Apply Kubernetes manifests
- [ ] Verify pods running
- [ ] Run smoke tests
- [ ] Check monitoring dashboards
