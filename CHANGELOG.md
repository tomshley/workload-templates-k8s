# Changelog

All notable changes to this project are documented in this file.

This project follows Semantic Versioning.

---

## [0.1.1] - 2026-03-29

### Added

- **Autoscaling components (2)** - HorizontalPodAutoscaler and Karpenter nodepool components
  - `hpa` - HorizontalPodAutoscaler templates for pod-level autoscaling
  - `karpenter-nodepool` - Karpenter EC2NodeClass and NodePool templates for dynamic node provisioning

- **Scaling example** - Complete example demonstrating HPA + Karpenter integration
  - `examples/scaling-hpa-karpenter/` - End-to-end scaling configuration
  - Environment-specific patches for EC2NodeClass configuration

- **Network load balancer component** - AWS NLB TCP Service with TLS termination
  - `service-nlb-tcp` - Internet-facing NLB template with ACM certificate placeholder

- **Enhanced kustomizeconfig** - Improved component composition support

### Changed

- **pekko-cluster example** - Updated to reference new autoscaling components
- **README.md** - Updated project structure to include connection components (from 0.1.0) and new components

---

## [0.1.0] - 2026-03-26

### Added

- **Connection components (5)** - Reusable components for common service connections
  - `connection-kafka` - Kafka connection with SASL credentials and config
  - `connection-postgres` - PostgreSQL connection with database config and credentials  
  - `connection-rds-cert` - RDS CA certificate bundle for TLS connections
  - `connection-s3` - S3 connection with bucket configuration
  - `credentials-registry` - GitLab container registry credentials

- **Component structure** - Each connection component includes:
  - `.env.example` files for local development reference
  - Secret and ConfigMap templates with PLACEHOLDER values
  - Kustomization configuration for easy consumption

### Changed

- **README.md** - Updated documentation to reflect new connection components
- **VERSION** - Bumped to 0.1.0

- **kustomizeconfig.yaml** - Added volume secret name transformation for Deployment, StatefulSet, and CronJob to support RDS certificate mounting

- **Component naming consistency** - Updated resource names across components for better alignment with dash-prefix naming convention

- **Service examples** - Updated pekko-cluster bootstrap examples to include new connection components as resources

---

## [0.0.3] - 2026-03-17

### Fixed

- **kustomizeconfig.yaml: `commonLabels` → `labels` section key** — Changed label fieldSpecs section key from `commonLabels:` to `labels:` to fix "conflicting fieldspecs" error when consumers use the Kustomize v5.x `labels:` transformer. The `commonLabels:` key conflicted with the newer transformer API. Requires Kustomize v5.4.0+. Older versions can revert to `commonLabels:` (commented in file).
- **kustomizeconfig.yaml: removed redundant `images` fieldSpecs** — The `images:` section duplicated Kustomize v5.x built-in defaults for Deployment, StatefulSet, and CronJob container/initContainer paths, causing "conflicting fieldspecs" when loaded via `configurations:`. Removed active declarations; commented-out block retained for Kustomize < v5.0.0 users.

---

## [0.0.2] - 2026-03-17

### Added

**Workload Templates (4)**
- `pekko-cluster` — Production-hardened Pekko Cluster Deployment
  - Zero-unavailable rolling updates (`maxUnavailable: 0`)
  - Topology spread constraints (prevent pod co-location)
  - Graceful shutdown coordination (`preStop` sleep 30s + `terminationGracePeriodSeconds: 60`)
  - Stabilization window (`minReadySeconds: 15`)
  - Quorum-safe replica count (`replicas: 3`)
  - Extended rollback history (`revisionHistoryLimit: 10`)
  - Container-aware JVM sizing (`MaxRAMPercentage=70%`)
- `deployment-http` — Stateless HTTP Deployment with topology spread and safe rolling updates
- `stateful-service` — Generic StatefulSet template with persistent volume claims
- `cron-job` — CronJob template with concurrency control (`concurrencyPolicy: Forbid`)

**Components (5)**
- `pdb` — PodDisruptionBudget with `maxUnavailable: 1`
- `rbac-pod-reader` — RBAC Role + RoleBinding for Kubernetes API pod discovery
- `service-headless-pekko-bootstrap` — Headless Service for DNS-based Pekko cluster bootstrapping
- `service-public-http` — Public-facing HTTP Service (ClusterIP)
- `serviceaccount` — ServiceAccount template

**Examples (4)**
- `image-pull-secret` — Strategic merge patch pattern for adding imagePullSecrets
- `pekko-cluster-dns-bootstrap` — DNS-based Pekko Cluster bootstrap pattern
- `pekko-cluster-kubernetes-api-bootstrap` — Kubernetes API-based Pekko Cluster bootstrap pattern
- `service-consumer` — Complete remote consumption pattern with overlay examples

**Infrastructure**
- `kustomizeconfig.yaml` — Name reference and image transformation mappings for safe `namePrefix`/`nameSuffix` usage
- GitLab CI validation for all workload templates, components, and local examples
- Comprehensive README with remote consumption patterns and resource customization examples

**OSS Hygiene**
- LICENSE (Apache 2.0)
- NOTICE with trademark information
- SECURITY policy
- CONTRIBUTING guidelines
- CODE_OF_CONDUCT
- CHANGELOG
- ROADMAP

### Design Notes

- **Label-driven identity model** — Service identity is driven by consumer-defined `app` label via Kustomize's `labels` transformer with `includeSelectors: true`, not by resource names. Templates use short default name `app` that `namePrefix` transforms into clean resource names. `PLACEHOLDER_IMAGE` is the only remaining placeholder (works natively with Kustomize `images:` transformer).
- **APP_LABEL via Downward API** — `APP_LABEL` env var in pekko-cluster uses `fieldRef: metadata.labels['app']` instead of a hardcoded value, eliminating identity drift at runtime.
- **topologySpreadConstraints label propagation** — `kustomizeconfig.yaml` has `labels` fieldSpecs for `topologySpreadConstraints/labelSelector/matchLabels` on Deployment and StatefulSet. Without this, consumer labels do not propagate to topology spread constraints (not in Kustomize's default label injection paths).
- **actorSystemName isolation** — Pekko cluster `actorSystemName` label kept in separate labels block with `includeSelectors: false` to avoid coupling discovery label to Service/PDB selectors.
- **imagePullSecrets pattern** — Templates omit `imagePullSecrets` by default. Consumers add via strategic merge patch. See `examples/image-pull-secret/` for the pattern.
- **managed-by label selector decoupling** — `app.kubernetes.io/managed-by: kustomize` label uses `includeSelectors: false` / `includeTemplates: true` to keep it out of immutable selectors while preserving observability on pod templates. This avoids forward-compatibility issues if the label changes in future template versions.

### Features

- **Remote consumption** — Templates designed for consumption via GitLab/GitHub URLs with version pinning
- **Kubernetes correctness** — Startup/readiness/liveness probes, lifecycle hooks, topology spread, safe rolling updates
- **JVM resource model** — CPU requests, memory limits, NO CPU limits (avoid throttling)
- **Kustomize v5.3.0 compatibility** — Uses `labels` with `includeSelectors: true` (not deprecated `commonLabels`), no cross-directory `configurations` references
- **Extension points** — Service-specific configuration via Kustomize patches (env vars, volumes, resource overrides)

### Documentation

- Remote consumption patterns (GitLab/GitHub URL formats)
- Label injection behavior with selector coupling warnings
- ServiceAccount binding requirements
- Resource customization via Kustomize patches (replicas, memory, storage examples)
- Pekko cluster configuration (split-brain resolver guidance)

---
