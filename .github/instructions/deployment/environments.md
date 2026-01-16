# Environment Deployment - MAIP

## Environments

| Environment | AWS Profile | Region(s) | Purpose |
|-------------|-------------|-----------|---------|
| dev | maip-dev | us-east-1 | Development |
| qa | maip-qa | us-east-1 | Testing |
| staging | maip-staging | us-east-1, us-west-2 | Pre-production |
| prod | maip-prod | us-east-1, us-west-2 | Production |

## Deploy Dev Environment

```bash
maip-dev

# 1. Infrastructure (if not exists)
aws cloudformation deploy \
  --template-file DevOps/Cloud/AWS/cloudformation/networking/vpc.yaml \
  --stack-name maip-dev-networking \
  --parameter-overrides file://parameters/dev/networking.json

# 2. Configure kubectl
aws eks update-kubeconfig --name maip-dev-eks-cluster

# 3. Deploy all services
kubectl apply -f DevOps/Kubernetes/namespaces/maip-namespace.yaml
kubectl apply -f DevOps/Kubernetes/services/ -n maip-services

# 4. Verify
kubectl get pods -n maip-services
```

## Deploy QA Environment

```bash
maip-qa

# Deploy services
aws eks update-kubeconfig --name maip-qa-eks-cluster
kubectl apply -f DevOps/Kubernetes/services/ -n maip-services

# Run smoke tests
./scripts/smoke-test.sh qa
```

## Deploy Staging Environment

```bash
maip-staging

# Primary region
aws eks update-kubeconfig --name maip-staging-eks-cluster --region us-east-1
kubectl apply -f DevOps/Kubernetes/services/ -n maip-services

# DR region
maip-staging && export AWS_REGION=us-west-2
aws eks update-kubeconfig --name maip-staging-eks-cluster --region us-west-2
kubectl apply -f DevOps/Kubernetes/services/ -n maip-services
```

## Deploy Production

```bash
# ALWAYS use GitHub Actions for prod deployments
# Manual steps for emergency only:

maip-prod

# 1. Primary region
aws eks update-kubeconfig --name maip-prod-eks-cluster --region us-east-1
kubectl apply -f DevOps/Kubernetes/services/ -n maip-services

# 2. Verify primary
kubectl get pods -n maip-services
./scripts/smoke-test.sh prod

# 3. DR region
export AWS_REGION=us-west-2
aws eks update-kubeconfig --name maip-prod-eks-cluster --region us-west-2
kubectl apply -f DevOps/Kubernetes/services/ -n maip-services
```

## Deploy Single Service

```bash
# Build image
cd backend/microservices/{service-name}
docker build -t maip/{service-name}:${VERSION} .

# Push to ECR
aws ecr get-login-password | docker login --username AWS --password-stdin ${AWS_ACCOUNT}.dkr.ecr.us-east-1.amazonaws.com
docker tag maip/{service-name}:${VERSION} ${AWS_ACCOUNT}.dkr.ecr.us-east-1.amazonaws.com/maip/{service-name}:${VERSION}
docker push ${AWS_ACCOUNT}.dkr.ecr.us-east-1.amazonaws.com/maip/{service-name}:${VERSION}

# Update deployment
kubectl set image deployment/{service-name} {service-name}=${AWS_ACCOUNT}.dkr.ecr.us-east-1.amazonaws.com/maip/{service-name}:${VERSION} -n maip-services

# Verify rollout
kubectl rollout status deployment/{service-name} -n maip-services
```

## Rollback

```bash
# View history
kubectl rollout history deployment/{service-name} -n maip-services

# Rollback to previous
kubectl rollout undo deployment/{service-name} -n maip-services

# Rollback to specific revision
kubectl rollout undo deployment/{service-name} -n maip-services --to-revision={revision}
```

## Health Checks

```bash
# All pods healthy?
kubectl get pods -n maip-services

# Service endpoints
kubectl get svc -n maip-services

# Check specific service health
curl https://api.maip.example.com/api/v1/{service}/actuator/health
```
