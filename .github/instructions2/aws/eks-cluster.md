# EKS Cluster Management - MAIP

## Cluster Names
- `maip-dev-eks-cluster`
- `maip-qa-eks-cluster`
- `maip-staging-eks-cluster`
- `maip-prod-eks-cluster`

## Configure kubectl

```bash
# Dev
maip-dev && aws eks update-kubeconfig --name maip-dev-eks-cluster

# QA
maip-qa && aws eks update-kubeconfig --name maip-qa-eks-cluster

# Staging
maip-staging && aws eks update-kubeconfig --name maip-staging-eks-cluster

# Prod
maip-prod && aws eks update-kubeconfig --name maip-prod-eks-cluster
```

## Verify Connection

```bash
kubectl get nodes
kubectl get namespaces
```

## MAIP Namespace

```bash
# All MAIP services run in maip-services namespace
kubectl get pods -n maip-services
kubectl get services -n maip-services
kubectl get deployments -n maip-services
```

## Scale Deployment

```bash
# Scale specific service
kubectl scale deployment {service-name} -n maip-services --replicas={count}

# Examples:
kubectl scale deployment order-service -n maip-services --replicas=8
kubectl scale deployment pricing-engine-service -n maip-services --replicas=4
```

## Check HPA Status

```bash
kubectl get hpa -n maip-services
kubectl describe hpa {service-name}-hpa -n maip-services
```

## View Logs

```bash
# Tail logs for a service
kubectl logs -f -l app={service-name} -n maip-services --tail=100

# Examples:
kubectl logs -f -l app=order-service -n maip-services --tail=100
kubectl logs -f -l app=merchant-application-service -n maip-services --tail=50
```

## Emergency: Restart Deployment

```bash
kubectl rollout restart deployment/{service-name} -n maip-services
kubectl rollout status deployment/{service-name} -n maip-services
```

## Scale Node Group

```bash
# Check current node group
aws eks describe-nodegroup \
  --cluster-name maip-{env}-eks-cluster \
  --nodegroup-name maip-{env}-nodegroup

# Scale node group
aws eks update-nodegroup-config \
  --cluster-name maip-{env}-eks-cluster \
  --nodegroup-name maip-{env}-nodegroup \
  --scaling-config minSize=2,maxSize=20,desiredSize={count}
```

## Deploy KEDA (Event-Driven Autoscaling)

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace
```
