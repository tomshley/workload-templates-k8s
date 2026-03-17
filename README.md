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
  pekko-cluster-dns-bootstrap/            — DNS-based Pekko Cluster bootstrap
  pekko-cluster-kubernetes-api-bootstrap/ — Kubernetes API-based Pekko Cluster bootstrap
  service-consumer/                       — Remote consumption patterns

assets/
  brand/

kustomizeconfig.yaml — Kustomize nameReference/image transformation configuration
VERSION
```

---

## Usage

Templates are designed to be **consumed remotely** via Kustomize, not copied. Service repositories reference templates via GitLab/GitHub URLs and apply service-specific patches.

### Remote Consumption (Recommended)

See `examples/service-consumer/` for a complete example of how service repositories consume these templates:

```yaml
resources:
  - https://gitlab.com/your-org/workload-templates-k8s//workloads/pekko-cluster?ref=v1.0.0
  - https://gitlab.com/your-org/workload-templates-k8s//components/serviceaccount?ref=v1.0.0
```

Benefits:
- Version pinning via `ref=`
- No manifest copying
- Centralized template updates
- Clear dependency lineage

**Important:** When using `namePrefix` or `nameSuffix` with remote templates, also fetch `kustomizeconfig.yaml`:

```yaml
configurations:
  - https://gitlab.com/your-org/workload-templates-k8s//kustomizeconfig.yaml?ref=v1.0.0
```

This ensures cross-resource references (ServiceAccount, Service, Secret, Role) are automatically rewritten when names are transformed. Without it, `namePrefix` may rename resources but leave internal references pointing to the old names, causing runtime failures.

### Composition Patterns

`examples/pekko-cluster-dns-bootstrap/` and `examples/pekko-cluster-kubernetes-api-bootstrap/` demonstrate how workloads compose with components for specific use cases (DNS-based vs. Kubernetes API-based cluster discovery).

---

## Placeholder Consistency

Templates use placeholders that **must remain consistent** across composed resources:

- **`PLACEHOLDER_APP_NAME`** — Used in labels (`app`, `actorSystemName`), container names, and resource names
- **`PLACEHOLDER_SERVICE_ACCOUNT`** — Must match between `serviceAccountName` in workloads and `metadata.name` in serviceaccount component
- **`PLACEHOLDER_IMAGE_PULL_SECRET`** — Must match your registry credentials secret name
- **`PLACEHOLDER_IMAGE`** — Container image reference

When composing workloads + components, ensure these placeholders are replaced with the same values via Kustomize overlays or direct substitution.

### Important: Label Injection Behavior

Templates include `labels` with `includeSelectors: true` in their kustomization.yaml files:

```yaml
labels:
  - pairs:
      app.kubernetes.io/managed-by: kustomize
    includeSelectors: true
```

Kustomize's `labels` with `includeSelectors` injects labels into **both** `metadata.labels` and `spec.selector.matchLabels` (for Deployment, StatefulSet, PDB). This ensures consistent labeling but creates coupling:

- If you override or omit these labels in your overlay, selectors may not match pods
- Consumer overlays should **merge** with template labels, not replace them

This is intentional for observability and consistent resource labeling across the platform.

### Critical: ServiceAccount Binding

The workload's `serviceAccountName: PLACEHOLDER_SERVICE_ACCOUNT` **must match** the ServiceAccount component's `metadata.name: PLACEHOLDER_SERVICE_ACCOUNT`.

If you override the ServiceAccount name in your overlay, you **must also** patch the workload's `serviceAccountName` field to match.

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
      name: PLACEHOLDER_APP_NAME
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
