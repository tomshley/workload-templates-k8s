# Scaling Example: HPA + Karpenter NodePool (GCP-specific)

This example is **GCP-specific** — it composes the provider-neutral `hpa`
component with the opt-in `karpenter-nodepool-gcp` component (GCENodeClass
requires GKE Standard with the GCP Karpenter provider). On other substrates,
consume `hpa` alone and use your platform's node autoscaling; see the
Portability section of the root README.

It demonstrates **horizontal pod autoscaling with just-in-time node provisioning** using:

- **HPA** — scales pod replicas based on CPU utilization (70% target)
- **Karpenter NodePool + GCENodeClass** — provisions new GCE instances when pending pods can't be scheduled

## How It Works

1. HPA monitors CPU utilization across pods
2. When utilization exceeds 70%, HPA increases the replica count
3. If the cluster has insufficient capacity, pods enter `Pending` state
4. Karpenter detects pending pods and provisions new GCE instances matching the NodePool constraints
5. New pods are scheduled on the provisioned instances
6. When load decreases, HPA scales down replicas; Karpenter consolidates underutilized instances

## Prerequisites

- **Karpenter GCP provider controller** installed in a **GKE Standard** cluster — the GCP
  Karpenter provider targets GKE Standard, not Autopilot
- **Controller IAM** per the provider's `deploy/iam/karpenter-controller-role.yaml`
- **Dedicated node service account** (`roles/container.nodeServiceAccount`) — patched in per
  environment

## Required Patches

The `GCENodeClass` template uses a PLACEHOLDER that **must** be patched per environment:

| Placeholder | Description | Example |
|---|---|---|
| `PLACEHOLDER_NODE_SERVICE_ACCOUNT@PLACEHOLDER_PROJECT.iam.gserviceaccount.com` | Dedicated node service account email | `my-service-nodes@my-project.iam.gserviceaccount.com` |

Patch in your environment overlay:

```yaml
patches:
  - path: patches/gcenodeclass-env.yaml
    target:
      group: karpenter.k8s.gcp
      version: v1alpha1
      kind: GCENodeClass
      name: my-service-default
```

## Customization

The instance-family requirement and the boot disk are a matched pair — N4-generation families
take Hyperdisk only and reject every `pd-*` category, so an overlay that changes one side must
change the other:

```yaml
# production — N4 families on Hyperdisk
patches:
  - target:
      group: karpenter.sh
      version: v1
      kind: NodePool
      name: my-service-default
    patch: |-
      - op: replace
        path: /spec/template/spec/requirements/1
        value:
          key: karpenter.k8s.gcp/instance-family
          operator: In
          values: ["n4", "n4d"]
  - target:
      group: karpenter.k8s.gcp
      version: v1alpha1
      kind: GCENodeClass
      name: my-service-default
    patch: |-
      - op: replace
        path: /spec/disks/0/category
        value: hyperdisk-balanced
```

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
kustomize build --load-restrictor LoadRestrictionsNone examples/scaling-hpa-karpenter-gcp
```

## Infrastructure (Terraform)

The Karpenter controller and its prerequisites must exist before this example
is useful. How you provision them is up to you — for example, with Terraform
modules shaped like:

```hcl
module "karpenter_prereqs" {
  source = "root-modules-tf//terraform/modules/gcp-karpenter-prereqs"
  # Creates: least-privilege controller role, controller service account
  #          (via gcp-gke-workload-identity), actAs on the node account
}

module "karpenter_controller" {
  source = "root-modules-tf//terraform/modules/gcp-gke-karpenter-controller"
  # Installs: provider CRD + controller charts; sets the node service
  #           account as the controller default
}
```

Feed the outputs into `patches/gcenodeclass-env.yaml`:

- `spec.serviceAccount` ← `module.karpenter_prereqs.node_service_account_email`
- `spec.networkTags` ← `module.karpenter_controller.node_class_network_tags`
  (the controller module documents that this list is used verbatim in every
  NodeClass; omit the field when the list is empty)
