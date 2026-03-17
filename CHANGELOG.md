# Changelog

All notable changes to this project are documented in this file.

This project follows Semantic Versioning.

---

## [0.0.1] - 2026-03-17

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

**Examples (3)**
- `pekko-cluster-dns-bootstrap` — DNS-based Pekko Cluster bootstrap pattern
- `pekko-cluster-kubernetes-api-bootstrap` — Kubernetes API-based Pekko Cluster bootstrap pattern
- `service-consumer` — Complete remote consumption pattern with overlay examples

**Infrastructure**
- `kustomizeconfig.yaml` — Name reference and image transformation mappings for safe `namePrefix`/`nameSuffix` usage
- GitLab CI validation for all workload templates, components, and local examples
- Comprehensive README with remote consumption patterns, placeholder consistency warnings, and resource customization examples

**OSS Hygiene**
- LICENSE (Apache 2.0)
- NOTICE with trademark information
- SECURITY policy
- CONTRIBUTING guidelines
- CODE_OF_CONDUCT
- CHANGELOG
- ROADMAP

### Features

- **Remote consumption** — Templates designed for consumption via GitLab/GitHub URLs with version pinning
- **Kubernetes correctness** — Startup/readiness/liveness probes, lifecycle hooks, topology spread, safe rolling updates
- **JVM resource model** — CPU requests, memory limits, NO CPU limits (avoid throttling)
- **Placeholder consistency** — `PLACEHOLDER_APP_NAME`, `PLACEHOLDER_SERVICE_ACCOUNT`, `PLACEHOLDER_IMAGE`, `PLACEHOLDER_IMAGE_PULL_SECRET` used consistently across all resources
- **Kustomize v5.3.0 compatibility** — Uses `labels` with `includeSelectors: true` (not deprecated `commonLabels`), no cross-directory `configurations` references
- **Extension points** — Service-specific configuration via Kustomize patches (env vars, volumes, resource overrides)

### Documentation

- Remote consumption patterns (GitLab/GitHub URL formats)
- Placeholder consistency requirements
- Label injection behavior with selector coupling warnings
- ServiceAccount binding requirements
- Resource customization via Kustomize patches (replicas, memory, storage examples)
- Pekko cluster configuration (split-brain resolver guidance)

---
