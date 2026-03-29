# Scaling Example: HPA + Karpenter NodePool

This example demonstrates **horizontal pod autoscaling with just-in-time node provisioning** using:

- **HPA** — scales pod replicas based on CPU utilization (70% target)
- **Karpenter NodePool + EC2NodeClass** — provisions new EC2 nodes when pending pods can't be scheduled

## How It Works

1. HPA monitors CPU utilization across pods
2. When utilization exceeds 70%, HPA increases the replica count
3. If the cluster has insufficient capacity, pods enter `Pending` state
4. Karpenter detects pending pods and provisions new nodes matching the NodePool constraints
5. New pods are scheduled on the provisioned nodes
6. When load decreases, HPA scales down replicas; Karpenter consolidates underutilized nodes

## Prerequisites

- **Karpenter controller** installed in the cluster (via Terraform/Helm)
- **IAM roles** for Karpenter controller and node instances
- **Discovery tags** on VPC subnets and security groups: `karpenter.sh/discovery: <cluster-name>`
- **EKS access entry** for Karpenter node role (type: `EC2_LINUX`)

## Required Patches

The `EC2NodeClass` template uses PLACEHOLDERs that **must** be patched per environment:

| Placeholder | Description | Example |
|---|---|---|
| `PLACEHOLDER_KARPENTER_NODE_ROLE` | IAM role name for Karpenter nodes | `my-cluster-karpenter-node` |
| `PLACEHOLDER_CLUSTER_NAME` | EKS cluster name (subnet/SG discovery) | `my-cluster` |

Patch in your environment overlay:

```yaml
patches:
  - path: patches/ec2nodeclass-env.yaml
    target:
      group: karpenter.k8s.aws
      version: v1
      kind: EC2NodeClass
      name: my-service-default
```

## HPA Customization

Override HPA settings per environment:

```yaml
# staging — single replica, no scaling
patches:
  - target:
      kind: HorizontalPodAutoscaler
      name: my-service-app
    patch: |-
      - op: replace
        path: /spec/minReplicas
        value: 1
      - op: replace
        path: /spec/maxReplicas
        value: 1
```

```yaml
# production — scale 3–7 replicas
patches:
  - target:
      kind: HorizontalPodAutoscaler
      name: my-service-app
    patch: |-
      - op: replace
        path: /spec/minReplicas
        value: 3
      - op: replace
        path: /spec/maxReplicas
        value: 7
```

For Pekko Cluster with `keep-majority` SBR, production `minReplicas` must be **≥ 3** (odd).

## Testing

```bash
kustomize build examples/scaling-hpa-karpenter
```

## Infrastructure (Terraform)

The Karpenter controller and prerequisites are provisioned via Terraform modules:

```hcl
module "karpenter_prereqs" {
  source = "root-modules-tf//terraform/modules/aws-eks-karpenter-prereqs"
  # Creates: IAM roles, instance profile, SQS queue, EventBridge rules
}

module "karpenter_controller" {
  source = "root-modules-tf//terraform/modules/aws-eks-karpenter-controller"
  # Installs: Karpenter Helm chart with IRSA binding
}

resource "aws_eks_access_entry" "karpenter_node" {
  cluster_name  = module.cluster.cluster_name
  principal_arn = module.karpenter_prereqs.node_role_arn
  type          = "EC2_LINUX"
}
```
