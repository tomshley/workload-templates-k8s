<p>
  <img src="assets/brand/logo.svg" alt="Tomshley Logo" width="200"/>
</p>

# Tomshley Workload Templates for Kubernetes

Composable, opinionated Kubernetes manifest templates for workloads and shared components.

This repository is part of the **Tomshley – OSS IP Division** and is maintained by **Tomshley LLC**.
It provides reusable Kubernetes templates consumed by application repositories for consistent, production-ready deployments.

---

## Overview

Tomshley Workload Templates provides a small, opinionated set of **Kubernetes manifest templates** designed to act as stable foundations for:

- Application workloads (Deployments, StatefulSets, CronJobs)
- Shared infrastructure components (Services, PDBs, RBAC, ServiceAccounts, NetworkPolicies)
- Reference examples (e.g. Pekko Cluster bootstrapping patterns)

The project prioritizes correctness, minimalism, composability, and long-term maintainability.

---

## Features

- Composable workload and component templates
- Clear separation between workloads, components, and examples
- Production-ready defaults (CPU requests, memory limits, health checks, probes)
- Pekko Cluster bootstrapping patterns (DNS and Kubernetes API)
- OSS- and enterprise-friendly licensing (Apache 2.0)

---

## Architecture

The repository is organized around two orthogonal axes:

1. **Workloads** — complete deployment patterns (Deployment, StatefulSet, CronJob)
2. **Components** — reusable supporting resources (Services, PDBs, RBAC, ServiceAccounts, NetworkPolicies)

Workloads compose components. Examples show how workloads and components combine for specific use cases.

### Workload Selection

`deployment-http` is **runtime-neutral** — no language-specific env vars, a single `http` port, probes targeting that port. Suitable for any HTTP service runtime (JVM, Python, Go, Node, …). JVM consumers without Pekko Cluster requirements add `JAVA_TOOL_OPTIONS` via a strategic-merge patch in their overlay (see [`CHANGELOG.md`](./CHANGELOG.md) for the patch recipe).

`deployment-worker` is a **runtime-neutral, non-HTTP** long-running Deployment.
It defaults to one replica with `Recreate`, exposes no ports or synthetic
health endpoint, and relies on process exit for baseline liveness. Use it for
queue consumers, pollers, reconcilers, and other continuously running workers;
patch both replica count and update strategy when concurrent instances are
known to be safe. These rollout defaults reduce planned overlap; they do not
provide mutual exclusion. Work that requires a single owner must use an
external lease, partition assignment, or fencing token.

`pekko-cluster` is a **JVM/Pekko specialization** that bundles the defaults a Pekko Cluster service needs out of the box: JVM heap tuning, the Pekko Management port (7626), the remoting port (7355), Downward-API-driven `APP_LABEL` for contact-point discovery, and a longer cluster-leave grace period. Use it whenever your service participates in a Pekko Cluster.

`stateful-service` is a **runtime-neutral** StatefulSet with a longer preStop grace period and Downward-API pod-identity env (`POD_NAME`, `POD_NAMESPACE`). Probes follow the repo-wide `/alive` + `/ready` convention.

`cron-job` is a **runtime-neutral** CronJob template with `concurrencyPolicy: Forbid`. JVM consumers add `JAVA_TOOL_OPTIONS` via a strategic-merge patch in their overlay (see [`CHANGELOG.md`](./CHANGELOG.md) for the patch recipe; note the deeper `spec.jobTemplate.spec.template.spec.containers` path that `CronJob` requires).

### Probe Convention

All three HTTP-serving workloads (`deployment-http`, `pekko-cluster`, `stateful-service`) expose the same two probe **paths**:

- **`/alive`** — liveness; the process is alive. Always returns 200 for a running pod.
- **`/ready`** — readiness; the pod's dependencies are ready and it can accept traffic. Returns 200 only when readiness gates pass; 5xx otherwise.

Probe **ports** differ by workload: `deployment-http` and `stateful-service` target `port: http` (the application port); `pekko-cluster` targets `port: management` (Pekko Management's 7626, separate from the remoting and application data planes). Consumers that want to split probes off the application port on `deployment-http` or `stateful-service` can patch the `port:` field without changing paths.

The convention originates from Pekko Management's `HealthCheckRoutes` and is implementable by any HTTP service with two trivial routes. The JVM side ships out-of-the-box via `tomshley/boilerplate-jvm`; any HTTP framework on any runtime can serve the same two paths in a few lines of handler code.

---

## Project Structure

```
workloads/
  cron-job/               — CronJob template
  deployment-http/        — HTTP Deployment template
  deployment-worker/      — Non-HTTP long-running worker Deployment
  pekko-cluster/          — Pekko Cluster Deployment template
  stateful-service/       — Generic StatefulSet template

components/
  connection-db-ca-cert/   — Database CA bundle Secret (PostgreSQL, YugabyteDB, RDS, …)
  connection-kafka/         — Kafka connection Secret
  connection-postgres/     — PostgreSQL connection Secret/ConfigMap
  connection-s3/           — S3-compatible object storage ConfigMap
  credentials-registry/    — Container registry credentials Secret
  hpa/                    — HorizontalPodAutoscaler
  karpenter-nodepool/     — Karpenter NodePool policy (abstract base for provider wrappers)
  karpenter-nodepool-aws/ — AWS wrapper: EC2NodeClass + nodeClassRef binding
  karpenter-nodepool-gcp/ — GCP wrapper: GCENodeClass + nodeClassRef binding
  networkpolicy-default-deny-ingress/  — Deny-all ingress NetworkPolicy baseline for the namespace
  networkpolicy-pekko-cluster-peering/ — Pekko remoting/management allow between same-app pods
  pdb/                    — PodDisruptionBudget
  rbac-pod-reader/        — RBAC Role + RoleBinding for pod discovery
  server-tls/             — Opt-in Deployment mount of an externally managed TLS Secret
  service-headless-pekko-bootstrap/ — Headless Service for Pekko cluster bootstrapping
  service-public-http/    — Public-facing HTTP Service
  service-tcp-loadbalancer/     — TCP LoadBalancer Service (provider-neutral)
  service-tcp-loadbalancer-aws/ — AWS wrapper: NLB + ACM TLS annotations
  serviceaccount/         — ServiceAccount

examples/
  image-pull-secret/                      — imagePullSecrets patch pattern
  pekko-cluster-dns-bootstrap/            — DNS-based Pekko Cluster bootstrap
  pekko-cluster-kubernetes-api-bootstrap/ — Kubernetes API-based Pekko Cluster bootstrap
  pekko-cluster-network-isolation/        — Default-deny ingress + Pekko cluster peering allow
  scaling-hpa-karpenter-aws/              — HPA + Karpenter NodePool autoscaling (AWS-specific)
  scaling-hpa-karpenter-gcp/              — HPA + Karpenter NodePool autoscaling (GCP-specific)
  server-tls/                             — TLS file mounting with an external Secret
  service-consumer/                       — Remote consumption patterns
  worker-consumer/                        — Non-HTTP worker Deployment + ServiceAccount composition

assets/
  brand/

kustomizeconfig.yaml — Kustomize nameReference, label fieldSpecs, and image transformation configuration
VERSION
```

---

## Portability

The core of this library is **provider-neutral by contract**:

- **Workloads and unpostfixed components target any conformant Kubernetes** —
  k3s, kind, bare-metal kubeadm, and every managed offering alike. They use
  only stable core APIs (`apps/v1`, `autoscaling/v2`, `policy/v1`,
  `networking.k8s.io/v1`, RBAC) and never require a cloud-specific
  controller, CRD, storage class, or annotation to render and run.
- **Provider-specific behavior ships as a thin, postfixed wrapper over a
  neutral base.** `service-tcp-loadbalancer-aws` adds AWS NLB/ACM annotations
  to the neutral `service-tcp-loadbalancer`; `karpenter-nodepool-aws` binds
  the neutral `karpenter-nodepool` policy base to the AWS `EC2NodeClass`, and
  `karpenter-nodepool-gcp` binds the same base to the GCP `GCENodeClass`.
  Leaving a provider means swapping the wrapper path for the base's (or
  another provider's wrapper) — the base and everything layered on it stay
  put. Wrappers are opt-in leaves: nothing in `workloads/`, in a neutral
  component, or in `kustomizeconfig.yaml` depends on them for correct
  rendering.
- **Bases that cannot deploy alone fail closed.** `karpenter-nodepool` is an
  abstract base: Karpenter always needs a provider NodeClass, so the base's
  `nodeClassRef` is a placeholder the API server rejects — the same guard-rail
  idea as the leading-hyphen names and `PLACEHOLDER_IMAGE`.
- **Connection components are protocol- or product-scoped, never
  provider-scoped** — `connection-s3` configures any S3-compatible object
  store (MinIO, Ceph RGW, AWS S3, …); `connection-db-ca-cert` carries any
  database CA bundle (PostgreSQL, YugabyteDB, CloudNativePG, AWS RDS, …).

Contributions adding provider-coupled behavior are welcome under the same
rule: keep the neutral base free of provider fields, put provider bindings
in a `-<provider>` wrapper, and never make a neutral template depend on a
provider-postfixed one.

---

## Usage

Templates are designed to be **consumed remotely** via Kustomize, not copied. Service repositories reference templates via GitLab/GitHub URLs and apply service-specific patches.

### Remote Consumption (Recommended)

See `examples/service-consumer/` for a complete example of how service repositories consume these templates:

```yaml
resources:
  - https://gitlab.com/your-org/workload-templates-k8s//workloads/pekko-cluster?ref=v1.3.0
  - https://gitlab.com/your-org/workload-templates-k8s//components/serviceaccount?ref=v1.3.0
```

Benefits:
- Version pinning via `ref=`
- No manifest copying
- Centralized template updates
- Clear dependency lineage

**Important:** When using `namePrefix` or `nameSuffix` with remote templates, also fetch `kustomizeconfig.yaml`:

```yaml
configurations:
  - https://gitlab.com/your-org/workload-templates-k8s//kustomizeconfig.yaml?ref=v1.3.0
```

This ensures cross-resource references (ServiceAccount, Service, Secret, Role) are automatically rewritten when names are transformed. Without it, `namePrefix` may rename resources but leave internal references pointing to the old names, causing runtime failures.

### Composition Patterns

`examples/pekko-cluster-dns-bootstrap/` and `examples/pekko-cluster-kubernetes-api-bootstrap/` demonstrate how workloads compose with components for specific use cases (DNS-based vs. Kubernetes API-based cluster discovery).

---

## Identity Model

> **Labels define identity. Names are cosmetic.**

Templates use hyphen-prefixed default names (`-app`, `-kafka-credentials`, etc.) that Kustomize's `namePrefix` transforms into clean, production-ready resource names (e.g. `orders-app`). The leading hyphen acts as a **guard rail** — if a consumer forgets to set `namePrefix`, the resulting resource names (e.g. `-app`) will fail validation, making the omission obvious. Consumer `namePrefix` values should **not** include a trailing hyphen; the template's leading hyphen provides the separator. Service identity is driven by **labels**, not resource names.

### How It Works

Consumer overlays set the `app` label via Kustomize's `labels` transformer:

```yaml
labels:
  - pairs:
      app: orders
    includeSelectors: true
```

Kustomize propagates labels to:
- `metadata.labels` on all resources
- `spec.selector.matchLabels` on Deployments, StatefulSets, PDBs
- `spec.template.metadata.labels` on pod templates
- `spec.selector` on Services
- `spec.podSelector` and ingress peer `podSelector` on NetworkPolicies (an empty `podSelector: {}` stays empty)

This ensures consistent identity across all composed resources without manual patching.

### Best Practice

The `app` label **must be unique per deployment** in a namespace. Two services with `app: orders` in the same namespace will cross-select (PDBs target wrong pods, Services route to wrong backends, Pekko clusters cross-discover).

### `PLACEHOLDER_IMAGE`

`PLACEHOLDER_IMAGE` is the only placeholder that uses Kustomize's `images` transformer. Other components use annotation- or data-level `PLACEHOLDER_*` values that consumers override via patches.

```yaml
images:
  - name: PLACEHOLDER_IMAGE
    newName: registry.example.com/my-org/orders-server
    newTag: "1.0.0"
```

### ServiceAccount Binding

Workloads and the ServiceAccount component both use `app` as the default name. When `namePrefix` is applied, `kustomizeconfig.yaml` rewrites `serviceAccountName` references automatically. If you override the ServiceAccount name in your overlay, you **must also** patch the workload's `serviceAccountName` field to match.

### APP_LABEL Environment Variable (Pekko Cluster)

The `pekko-cluster` template exposes `APP_LABEL` as an env var for Pekko's contact-point discovery. This value is sourced from the pod's actual `app` label at runtime via the Downward API (`metadata.labels['app']`), so it always reflects the consumer's identity — no manual coordination required.

---

## Resource Customization

Template resource defaults — `cpu: 500m`, `memory: 1Gi` / `2Gi` across all five workloads and a `startupProbe.failureThreshold` window of 120–150s on the three HTTP-serving workloads — are sized for JVM-class services. Lower-footprint runtimes (Python, Go, Node) typically tighten both the resource floor and (where applicable) the startup-probe window in their overlay. Consumers should adjust based on workload characteristics:

- **replicas** — Scale for availability and throughput requirements
- **CPU requests** — Size for steady-state processing needs  
- **Memory limits** — Adjust for heap / runtime footprint (the `pekko-cluster` template bundles `-XX:InitialRAMPercentage=50 -XX:MaxRAMPercentage=70`; other workloads are runtime-neutral and leave heap sizing to the consumer overlay)
- **Storage requests** — StatefulSet PVC sizing for data volume

Override via Kustomize patches in your service overlay:

```yaml
patches:
  - target:
      kind: Deployment
      name: -app
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
      - op: replace
        path: /spec/template/spec/containers/0/resources/limits/memory
        value: "4Gi"
```

For StatefulSet storage:

```yaml
patches:
  - target:
      kind: StatefulSet
    patch: |-
      - op: replace
        path: /spec/volumeClaimTemplates/0/spec/resources/requests/storage
        value: "100Gi"
```

---

## Image Pull Secrets

Templates intentionally **omit** `imagePullSecrets` by default. If your registry requires authentication (e.g., private GitLab/GitHub Container Registry, Docker Hub private repos), add `imagePullSecrets` via a strategic merge patch in your overlay.

### Pattern

See `examples/image-pull-secret/` for the complete pattern:

```yaml
# examples/image-pull-secret/kustomization.yaml
resources:
  - ../../workloads/deployment-http
  - ../../components/serviceaccount

patches:
  - path: patches/deployment-image-pull-secret.yaml
    target:
      kind: Deployment
      name: -app
```

```yaml
# examples/image-pull-secret/patches/deployment-image-pull-secret.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: -app
spec:
  template:
    spec:
      imagePullSecrets:
        - name: my-registry-credentials
```

### CI/CD Integration

The secret name (`my-registry-credentials`) must match:
1. A pre-created Kubernetes Secret in your target namespace
2. The `K8S_IMAGE_PULL_SECRET` value in your `.secure_files/.env` (if using GitLab CI)

CI creates the Secret from registry credentials before deploying. Your Kustomize overlay references it.

---

## Server TLS Files

`components/server-tls` is an opt-in Kustomize Component for the `app` container in a Deployment named `-app`. It works with `deployment-http`, `deployment-worker`, and `pekko-cluster`; it does not change workloads unless included under `components:`.

The component mounts a pre-existing Secret named `server-tls` at `/etc/ssl/server`, read-only, projecting `tls.crt` and `tls.key`. Both keys are required. It creates no Secret, grants no API permissions, and installs no certificate controller. File ownership and access restrictions remain the workload owner's responsibility.

Use a consumer patch to select the actual Secret name, as shown in `examples/server-tls`. An external Secret is not renamed by `namePrefix`; its configured name must match a Secret in the workload's namespace. The example also retains an independent application-data mount.

The application must enable TLS and load the certificate chain and private key from those files. Mounting them does not enable TLS, HTTP/2, or certificate reload. A LoadBalancer carrying this listener must pass through TCP if the application terminates TLS.

Before deployment, the operator supplies a trusted certificate chain and matching private key. On renewal, the operator replaces the Secret and performs a controlled workload rollout unless the application explicitly supports reload. Retire old certificate material after the old pods have drained; stopping or removing the workload does not delete the externally owned Secret. A missing Secret or missing required key blocks pod startup rather than enabling plaintext.

Validate the composition without a cluster:

```sh
kustomize build examples/server-tls
```

---

## Network Isolation

Two components implement a standard ingress boundary using only the core
`networking.k8s.io/v1` NetworkPolicy API. They bind traffic on any cluster
whose network plugin enforces NetworkPolicy — Calico, Cilium (including GKE
Dataplane V2), or the AWS VPC CNI with network-policy enforcement enabled —
and have no effect on a cluster without one; the manifests still record the
intended flow.

- **`networkpolicy-default-deny-ingress`** selects every pod in the namespace
  (`podSelector: {}`) with `policyTypes: [Ingress]` and no rules: all inbound
  pod traffic is denied unless another policy reopens it. NetworkPolicies
  union, so each intended caller is restored by an additive allow — nothing
  is ever re-widened by accident. Egress is untouched: DNS, the Kubernetes
  API, databases, brokers, and outbound tunnels keep working. The
  NetworkPolicy API always admits connections from the pod's own node, so
  kubelet probes keep working.
- **`networkpolicy-pekko-cluster-peering`** reopens only the Pekko cluster
  data plane: remoting (7355) and management (7626) between pods in the same
  namespace carrying the workload's `app` label. Without it a default-deny
  leaves a `pekko-cluster` workload unable to form. The application request
  port is **not** opened here — caller identity is consumer policy; a
  consumer grants it with its own allow policy naming the caller's namespace
  and pod labels.

The default-deny is scoped to the namespace, not to the workload whose
overlay renders it. Include it in every overlay that relies on it: each
overlay's `namePrefix` gives its copy a distinct name, so the copies coexist
as separate objects carrying the same rule — their union is the same deny,
and deleting one workload never lifts the baseline from the others. Give
every overlay in a namespace its own `namePrefix`; two overlays rendering
the same name would share one object, and pruning either would delete it.

The deny also covers traffic that reaches pods through a Service. Callers
behind an in-cluster ingress controller or Gateway implementation are
admitted by selecting that controller's namespace and pods. Traffic from a
`LoadBalancer` Service arrives from client, load balancer, or node addresses
rather than from pods — which one depends on the load balancer and
`externalTrafficPolicy` — so its allow needs an `ipBlock` covering those
source ranges.

Selectors carry the same `app: app` placeholder as the Service selectors —
a consumer's `labels` transform with `includeSelectors: true` rewrites
`spec.podSelector` and `spec.ingress[].from[].podSelector` together, so the
peer rule always names the real workload label.
`examples/pekko-cluster-network-isolation/` composes both components with a
`pekko-cluster` workload under that transform.

---

## Pekko Cluster Configuration

The `pekko-cluster` template provides Kubernetes-level safety (rolling updates, pod distribution, graceful shutdown). **Application-level cluster safety** requires configuration in your service's `application.conf`:

```hocon
pekko.cluster.split-brain-resolver {
  active-strategy = keep-majority
}
```

The template's `replicas: 3` default supports majority-based split-brain resolution. For production Pekko clusters:

- Use **odd replica counts** (3, 5, 7) to enable clear majority determination
- Configure a **split-brain resolver** to handle network partitions
- The `keep-majority` strategy is recommended for Kubernetes deployments

See [Pekko Split Brain Resolver documentation](https://pekko.apache.org/docs/pekko/current/split-brain-resolver.html) for detailed configuration options.

### gRPC

The `pekko-cluster` Deployment declares container port `grpc`/9900 and the `-app-bootstrap` headless Service publishes it, so DNS returns one A record per pod — the endpoint set client-side balancing needs. Declaring a container port is informational and opens nothing, so workloads that serve no gRPC are unaffected.

The Service provides per-pod DNS only — it does not by itself load balance. Balancing requires a pekko-grpc client configured with `service-discovery.mechanism = "pekko-dns"` and `load-balancing-policy = "round_robin"`. The `static` and `grpc-dns` mechanisms resolve a single address per lookup (`GrpcClientSettings.fromConfig` maps both to `staticServiceDiscovery`, a single `ResolvedTarget`) and cannot balance. pekko-grpc has no `service-discovery.refresh-interval` key — re-resolution comes from grpc-java calling the resolver's `refresh()` on connection breakage, bounded by DNS TTL. A non-Pekko gRPC client must supply its own client-side load balancing.

The Service keeps `publishNotReadyAddresses: true` because DNS-based bootstrap needs not-ready addresses or formation deadlocks. The field is service-wide with no per-port control, so a consumer that serves request traffic from this Service AND does not rely on DNS-based bootstrap discovery should patch it to `false` in its own overlay — a pod listening on the gRPC port that has not passed readiness has not joined the cluster, and routing requests to it times out on its cluster calls rather than refusing fast. The `kubernetes-api` discovery method — the default — queries the Kubernetes API by label selector, does not use this Service, and can patch safely.

### PDB and Replica Count Interaction

The `pdb` component sets `maxUnavailable: 1`, which is safe with the default 3 replicas (majority of 3 = 2; losing 1 pod is survivable). **If you reduce replicas to 2**, the PDB becomes unsafe with `keep-majority` SBR:

- Majority of 2 = **2** (both must be up)
- PDB allows K8s to voluntarily disrupt 1 pod (e.g. node drain)
- Surviving pod has no quorum → SBR shuts it down → **complete cluster outage**

If you need 2 replicas, override the PDB to `maxUnavailable: 0` (blocks voluntary disruptions) or use a different SBR strategy.

### Single-Replica Clusters (Staging / Development)

Running 1 replica is safe for staging and development environments:

- SBR is irrelevant — a single-node cluster has no partitions to resolve
- `REQUIRED_CONTACT_POINT_NR=1` (the template default) allows a single pod to form a cluster
- PDB `maxUnavailable: 1` blocks voluntary evictions, preventing accidental scale-to-zero
- Pekko Cluster Bootstrap discovers zero peers and self-joins after the contact point timeout

Override replicas via the Kustomize `replicas` transformer in your staging overlay:

```yaml
replicas:
  - name: my-service-app
    count: 1
```

### HPA and Replica Management

The `hpa` component provides a `HorizontalPodAutoscaler` that scales the Deployment based on CPU utilization. When HPA manages replicas:

- **Do not set a static `replicas` field** on the Deployment — HPA becomes the source of truth
- Remove any `deployment-replicas.yaml` patch or `replicas` transformer from overlays where HPA is active
- Consumer overlays patch `minReplicas` / `maxReplicas` per environment
- For Pekko Cluster with `keep-majority`, production `minReplicas` must be **≥ 3** (odd)
- Staging can use `minReplicas: 1` (single-node cluster, SBR irrelevant)

The `kustomizeconfig.yaml` nameReference ensures `scaleTargetRef.name` is automatically rewritten when `namePrefix` is applied.

### Karpenter NodePool (neutral base + provider wrappers)

Karpenter splits across two components following the wrapper pattern from
[Portability](#portability):

- **`karpenter-nodepool`** — the provider-neutral policy base (`NodePool` is a
  core `karpenter.sh` kind): capacity type, resource limits, consolidation.
  It is **abstract**: its `nodeClassRef` is a fail-closed placeholder, because
  every Karpenter installation requires a provider NodeClass. Never consume it
  directly.
- **`karpenter-nodepool-aws`** — the AWS wrapper: supplies the `EC2NodeClass`,
  binds the base's `nodeClassRef` to it, and appends the EC2 instance-type
  requirement. When HPA scales pods beyond cluster capacity, Karpenter
  provisions new EC2 instances matching the NodePool constraints.
- **`karpenter-nodepool-gcp`** — the GCP wrapper: supplies the `GCENodeClass`
  (`karpenter.k8s.gcp/v1alpha1`, from the
  [GCP provider](https://github.com/cloudpilot-ai/karpenter-provider-gcp)),
  binds the base's `nodeClassRef` to it, and appends the GCE instance-family
  requirement. Note `karpenter.k8s.gcp/instance-family` matches the
  machine-type prefix (`n2`, `n2d`), not the full family+shape. The families
  and the `GCENodeClass` boot disk are a matched pair: the default
  `pd-balanced` suits N2/N2D, while N4-generation families take Hyperdisk
  only — change both in the same overlay. The node service account is a
  fail-closed placeholder patched per environment, like the AWS wrapper's
  `role`.

Other Karpenter providers (Azure, Cluster API, …) pair the same base
with their own NodeClass kind — sibling `karpenter-nodepool-<provider>`
wrappers are welcome contributions. Substrates without a Karpenter provider
use their platform's node autoscaling and consume the provider-neutral `hpa`
alone.

**Prerequisites (AWS wrapper):**
- Karpenter controller installed in the cluster (via Terraform/Helm)
- IAM roles for controller (IRSA) and node instances
- Discovery tags on VPC subnets and security groups: `karpenter.sh/discovery: <cluster-name>`
- EKS access entry for the Karpenter node role (type: `EC2_LINUX`)

**Required patches:** The EC2NodeClass template uses PLACEHOLDERs that must be replaced per environment:

```yaml
# In your environment overlay
patches:
  - path: patches/ec2nodeclass-env.yaml
    target:
      group: karpenter.k8s.aws
      version: v1
      kind: EC2NodeClass
      name: my-service-default  # namePrefix + "-default"
```

Where `patches/ec2nodeclass-env.yaml` replaces `PLACEHOLDER_KARPENTER_NODE_ROLE` and `PLACEHOLDER_CLUSTER_NAME` with real values from your infrastructure outputs.

**Prerequisites (GCP wrapper):**
- Karpenter GCP provider controller installed in a GKE Standard cluster, with the controller IAM from the provider's `deploy/iam/karpenter-controller-role.yaml`
- A dedicated least-privilege node service account (`roles/container.nodeServiceAccount`)

**Required patches:** The GCENodeClass template's `serviceAccount` is a PLACEHOLDER that must be replaced per environment:

```yaml
# In your environment overlay
patches:
  - path: patches/gcenodeclass-env.yaml
    target:
      group: karpenter.k8s.gcp
      version: v1alpha1
      kind: GCENodeClass
      name: my-service-default  # namePrefix + "-default"
```

Where `patches/gcenodeclass-env.yaml` sets `spec.serviceAccount` to the node service account email from your infrastructure outputs.

See `examples/scaling-hpa-karpenter-aws/` and `examples/scaling-hpa-karpenter-gcp/` for the complete pattern including HPA + Karpenter composition.

The `kustomizeconfig.yaml` nameReference ensures `nodeClassRef.name` in the NodePool is automatically rewritten when `namePrefix` is applied.

### Scaling from 1 → 3 Replicas

When scaling a single-replica Pekko cluster to 3 (e.g. promoting staging configuration to production):

- New pods join the existing cluster automatically via Pekko Cluster Bootstrap
- Cluster Sharding rebalances shards across the new members
- No manual seed node configuration required — the Kubernetes API or DNS discovery handles formation

### CPU Limits

Per [Akka](https://doc.akka.io/libraries/akka-core/current/additional/deploying.html) and [Pekko](https://pekko.apache.org/docs/pekko/current/additional/deploying.html) recommendations, templates intentionally **omit CPU limits** to avoid CFS scheduler throttling. Only `resources.requests.cpu` is set. Use `-XX:ActiveProcessorCount` to control JVM thread pool sizing when no CPU limit is present.

---

## Versioning

A top-level `VERSION` file is the single source of truth for project release metadata.

---

## Contributing

See CONTRIBUTING.md.

---

## Security

See SECURITY.md.

---

## License

Apache License 2.0. See LICENSE.md and NOTICE.md.

---

## Credits

Maintained by Tomshley LLC.
Tomshley and the Tomshley logo are trademarks of Tomshley LLC.

---
