# Changelog

All notable changes to this project are documented in this file.

This project follows Semantic Versioning.

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
- **topologySpreadConstraints label propagation** — `kustomizeconfig.yaml` has `commonLabels` fieldSpecs for `topologySpreadConstraints/labelSelector/matchLabels` on Deployment and StatefulSet. Without this, consumer labels do not propagate to topology spread constraints (not in Kustomize's default label injection paths).
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
