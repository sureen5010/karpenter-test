# Karpenter Installation with Kustomize + ArgoCD

## Directory Structure

```
karpenter/
├── base/
│   ├── kustomization.yaml       # Base Kustomize config
│   ├── namespace.yaml           # Karpenter namespace
│   └── values.yaml              # Common Helm values
└── shs-nextgen-stage/
    └── hsarch-eks-cluster/
        ├── kustomization.yaml   # Overlay config
        ├── override-values.yml  # Cluster-specific values
        ├── nodeclass.yaml       # EC2NodeClass CRD
        ├── nodepool.yaml        # NodePool CRD
        └── application.yaml     # ArgoCD Application
```

## Prerequisites

1. **EKS Cluster**: `hsarch-eks-cluster` running in `us-east-2`
2. **IAM Roles**:
   - Karpenter Controller: `arn:aws:iam::523140806637:role/Karpenter-hsarch-eks-cluster-oh`
   - Karpenter Node: `arn:aws:iam::523140806637:role/KarpenterNodeRole-hsarch-eks-cluster-oh`
3. **SQS Queue**: `hsarch-eks-cluster-oh` (for spot interruption)
4. **Tagged Resources**:
   - Subnets with tag: `karpenter.sh/discovery=hsarch-eks-cluster`
   - Security groups with tag: `karpenter.sh/discovery=hsarch-eks-cluster`
5. **ArgoCD**: Installed in the cluster

## Deployment Steps

### 1. Apply ArgoCD Application

```bash
kubectl apply -f karpenter/shs-nextgen-stage/hsarch-eks-cluster/application.yaml
```

### 2. Verify Deployment

```bash
# Check ArgoCD app status
argocd app get karpenter-hsarch-eks-cluster

# Check Karpenter pods
kubectl -n karpenter get pods

# Check CRDs
kubectl get nodepool
kubectl get ec2nodeclass
```

### 3. Test with Sample Workload

```bash
# Create test deployment
kubectl create deployment inflate --image=public.ecr.aws/eks-distro/kubernetes/pause:3.7
kubectl scale deployment inflate --replicas=5

# Watch Karpenter provision nodes
kubectl -n karpenter logs -l app.kubernetes.io/name=karpenter -f
kubectl get nodes -w
```

## Manual Kustomize Build (Optional)

```bash
# Preview what will be deployed
kubectl kustomize karpenter/shs-nextgen-stage/hsarch-eks-cluster

# Apply directly without ArgoCD
kubectl apply -k karpenter/shs-nextgen-stage/hsarch-eks-cluster
```

## Customization

- **Base values**: Edit `karpenter/base/values.yaml`
- **Cluster-specific**: Edit `karpenter/shs-nextgen-stage/hsarch-eks-cluster/override-values.yml`
- **Node requirements**: Edit `nodepool.yaml`
- **AMI/Security**: Edit `nodeclass.yaml`

## Troubleshooting

```bash
# Check Karpenter logs
kubectl -n karpenter logs -l app.kubernetes.io/name=karpenter --tail=100

# Check node provisioning events
kubectl get events -n karpenter --sort-by='.lastTimestamp'

# Describe NodePool
kubectl describe nodepool default
```