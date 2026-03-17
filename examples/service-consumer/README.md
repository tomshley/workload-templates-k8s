# Service Consumer Example

This example demonstrates **how service repositories consume workload templates remotely**.

## Purpose

The other examples (`pekko-cluster-dns-bootstrap`, `pekko-cluster-kubernetes-api-bootstrap`) show **composition patterns** — which components work together for specific use cases.

This example shows **consumption patterns** — how your service repository references and customizes these templates without copying manifests.

---

## Remote Reference Pattern

Service repos reference templates via GitLab/GitHub URLs:

```yaml
resources:
  - https://gitlab.com/your-org/workload-templates-k8s//workloads/pekko-cluster?ref=v0.0.2
```

**Pattern**: `{host}/{org}/{repo}//{path}?ref={version}`

Key benefits:
- No manifest copying
- Version pinning via `ref=`
- Centralized template updates
- Clear dependency lineage

---

## Required Customizations

Every service overlay **must** provide:

### 1. Namespace
```yaml
namespace: your-platform-namespace
```

### 2. Name Prefix
```yaml
namePrefix: egress-
```

Prevents resource name collisions when multiple services share a namespace.

### 3. Identity Labels
```yaml
labels:
  - pairs:
      app: egress
    includeSelectors: true
```

Labels define identity. The `app` label is the canonical selector/routing label — it must be unique per deployment in the namespace.

For Pekko clusters, add `actorSystemName` in a **separate** block to avoid coupling it to selectors:

```yaml
  - pairs:
      actorSystemName: egress
    includeSelectors: false
    includeTemplates: true
```

### 4. Image Substitution
```yaml
images:
  - name: PLACEHOLDER_IMAGE
    newName: registry.gitlab.com/your-org/your-service
    newTag: "1.0.0"
```

---

## Service-Specific Configuration

Service overlays typically add:

### Environment Variables
```yaml
patches:
  - path: patches/deployment-env.yaml
    target:
      kind: Deployment
      name: app
```

`patches/deployment-env.yaml`:
```yaml
- op: add
  path: /spec/template/spec/containers/0/env/-
  value:
    name: MESSAGE_BROKER_URL
    valueFrom:
      secretKeyRef:
        name: message-broker-credentials
        key: url
```

### Volume Mounts for TLS Certs
```yaml
- op: add
  path: /spec/template/spec/volumes/-
  value:
    name: database-tls
    secret:
      secretName: database-ca-cert

- op: add
  path: /spec/template/spec/containers/0/volumeMounts/-
  value:
    name: database-tls
    mountPath: /etc/ssl/database
    readOnly: true
```

### ConfigMaps
```yaml
- op: add
  path: /spec/template/spec/containers/0/envFrom/-
  value:
    configMapRef:
      name: service-config
```

---

## Directory Structure in Service Repo

```
your-service/
├── k8s/
│   ├── base/
│   │   ├── kustomization.yaml          # References remote templates
│   │   └── patches/
│   │       ├── deployment-env.yaml     # Service-specific env vars
│   │       └── deployment-volumes.yaml # TLS certs, secrets
│   ├── overlays/
│   │   ├── dev/
│   │   │   └── kustomization.yaml      # Dev namespace, dev image tag
│   │   ├── staging/
│   │   │   └── kustomization.yaml
│   │   └── prod/
│   │       └── kustomization.yaml
```

---

## Testing the Example

```bash
# Validate manifests
kustomize build examples/service-consumer

# Apply to cluster (dry-run)
kustomize build examples/service-consumer | kubectl apply --dry-run=client -f -

# Deploy
kustomize build examples/service-consumer | kubectl apply -f -
```

---

## Version Pinning Strategy

### Development
```yaml
?ref=main  # Track latest (not recommended for production)
```

### Staging
```yaml
?ref=v0.0.2  # Pin to specific release
```

### Production
```yaml
?ref=v0.0.2  # Always pin to tested version
```

Update production pins only after staging validation.

---

## GitLab vs GitHub URL Format

### GitLab
```yaml
https://gitlab.com/org/repo//path?ref=version
```

### GitHub
```yaml
https://github.com/org/repo//path?ref=version
```

Note the **double slash** (`//`) before the path in both formats.
