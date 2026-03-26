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
- Shared infrastructure components (Services, PDBs, RBAC, ServiceAccounts)
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
2. **Components** — reusable supporting resources (Services, PDBs, RBAC, ServiceAccounts)

Workloads compose components. Examples show how workloads and components combine for specific use cases.

---

## Project Structure

```
workloads/
  cron-job/               — CronJob template
  deployment-http/        — HTTP Deployment template
  pekko-cluster/          — Pekko Cluster Deployment template
  stateful-service/       — Generic StatefulSet template

components/
  pdb/                    — PodDisruptionBudget
  rbac-pod-reader/        — RBAC Role + RoleBinding for pod discovery
  service-headless-pekko-bootstrap/ — Headless Service for Pekko cluster bootstrapping
  service-public-http/    — Public-facing HTTP Service
  serviceaccount/         — ServiceAccount

examples/
  image-pull-secret/                      — imagePullSecrets patch pattern
  pekko-cluster-dns-bootstrap/            — DNS-based Pekko Cluster bootstrap
  pekko-cluster-kubernetes-api-bootstrap/ — Kubernetes API-based Pekko Cluster bootstrap
  service-consumer/                       — Remote consumption patterns

assets/
  brand/

kustomizeconfig.yaml — Kustomize nameReference, label fieldSpecs, and image transformation configuration
VERSION
```

---

## Usage

Templates are designed to be **consumed remotely** via Kustomize, not copied. Service repositories reference templates via GitLab/GitHub URLs and apply service-specific patches.

### Remote Consumption (Recommended)

See `examples/service-consumer/` for a complete example of how service repositories consume these templates:

```yaml
resources:
  - https://gitlab.com/your-org/workload-templates-k8s//workloads/pekko-cluster?ref=v0.1.0
  - https://gitlab.com/your-org/workload-templates-k8s//components/serviceaccount?ref=v0.1.0
```

Benefits:
- Version pinning via `ref=`
- No manifest copying
- Centralized template updates
- Clear dependency lineage

**Important:** When using `namePrefix` or `nameSuffix` with remote templates, also fetch `kustomizeconfig.yaml`:

```yaml
configurations:
  - https://gitlab.com/your-org/workload-templates-k8s//kustomizeconfig.yaml?ref=v0.1.0
```

This ensures cross-resource references (ServiceAccount, Service, Secret, Role) are automatically rewritten when names are transformed. Without it, `namePrefix` may rename resources but leave internal references pointing to the old names, causing runtime failures.

### Composition Patterns

`examples/pekko-cluster-dns-bootstrap/` and `examples/pekko-cluster-kubernetes-api-bootstrap/` demonstrate how workloads compose with components for specific use cases (DNS-based vs. Kubernetes API-based cluster discovery).

---

## Identity Model

> **Labels define identity. Names are cosmetic.**

Templates use hyphen-prefixed default names (`-app`, `-kafka-credentials`, etc.) that Kustomize's `namePrefix` transforms into clean, production-ready resource names (e.g. `egress-app`). The leading hyphen acts as a **guard rail** — if a consumer forgets to set `namePrefix`, the resulting resource names (e.g. `-app`) will fail validation, making the omission obvious. Consumer `namePrefix` values should **not** include a trailing hyphen; the template's leading hyphen provides the separator. Service identity is driven by **labels**, not resource names.

### How It Works

Consumer overlays set the `app` label via Kustomize's `labels` transformer:

```yaml
labels:
  - pairs:
      app: egress
    includeSelectors: true
```

Kustomize propagates labels to:
- `metadata.labels` on all resources
- `spec.selector.matchLabels` on Deployments, StatefulSets, PDBs
- `spec.template.metadata.labels` on pod templates
- `spec.selector` on Services

This ensures consistent identity across all composed resources without manual patching.

### Best Practice

The `app` label **must be unique per deployment** in a namespace. Two services with `app: egress` in the same namespace will cross-select (PDBs target wrong pods, Services route to wrong backends, Pekko clusters cross-discover).

### `PLACEHOLDER_IMAGE`

`PLACEHOLDER_IMAGE` is the only remaining placeholder. It works natively with Kustomize's `images` transformer:

```yaml
images:
  - name: PLACEHOLDER_IMAGE
    newName: registry.example.com/my-org/egress-server
    newTag: "1.0.0"
```

### ServiceAccount Binding

Workloads and the ServiceAccount component both use `app` as the default name. When `namePrefix` is applied, `kustomizeconfig.yaml` rewrites `serviceAccountName` references automatically. If you override the ServiceAccount name in your overlay, you **must also** patch the workload's `serviceAccountName` field to match.

### APP_LABEL Environment Variable (Pekko Cluster)

The `pekko-cluster` template exposes `APP_LABEL` as an env var for Pekko's contact-point discovery. This value is sourced from the pod's actual `app` label at runtime via the Downward API (`metadata.labels['app']`), so it always reflects the consumer's identity — no manual coordination required.

---

## Resource Customization

Template resource defaults are conservative starting points. Consumers should adjust based on workload characteristics:

- **replicas** — Scale for availability and throughput requirements
- **CPU requests** — Size for steady-state processing needs  
- **Memory limits** — Adjust for heap requirements (JVM templates use `MaxRAMPercentage=70%`)
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

### PDB and Replica Count Interaction

The `pdb` component sets `maxUnavailable: 1`, which is safe with the default 3 replicas (majority of 3 = 2; losing 1 pod is survivable). **If you reduce replicas to 2**, the PDB becomes unsafe with `keep-majority` SBR:

- Majority of 2 = **2** (both must be up)
- PDB allows K8s to voluntarily disrupt 1 pod (e.g. node drain)
- Surviving pod has no quorum → SBR shuts it down → **complete cluster outage**

If you need 2 replicas, override the PDB to `maxUnavailable: 0` (blocks voluntary disruptions) or use a different SBR strategy.

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
