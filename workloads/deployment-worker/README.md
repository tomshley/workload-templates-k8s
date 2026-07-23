# `deployment-worker`

A **runtime-neutral, non-HTTP, long-running worker** Deployment.

Use this for a process that runs continuously but **serves no HTTP traffic** —
a message consumer, a stream processor, a poller, or a reconciler. It is the
non-serving counterpart to `deployment-http`.

## Why it exists

The existing long-running workloads assume an HTTP server. The non-HTTP
`cron-job` is finite rather than continuously running:

| Workload            | Shape                          | Probes (`/alive` + `/ready`) | HTTP-serving |
| ------------------- | ------------------------------ | ---------------------------- | ------------ |
| `deployment-http`   | Deployment, RollingUpdate, N≥2 | httpGet on `http`            | yes          |
| `pekko-cluster`     | Deployment, JVM/Pekko          | httpGet on `management`      | yes          |
| `stateful-service`  | StatefulSet                    | httpGet on `http`            | yes          |
| `cron-job`          | CronJob (runs to completion)   | n/a                          | no           |
| **`deployment-worker`** | **Deployment, Recreate, N=1** | **none (process liveness)** | **no**     |

A worker that has no HTTP listener cannot satisfy `httpGet` probes, and a
long-running consumer is not a `cron-job` (it never completes). That gap is what
this template fills.

## Defaults and the rationale

- **`replicas: 1` + `strategy: Recreate`.** This conservative baseline avoids
  deliberate old/new pod overlap during a Deployment rollout. It does **not**
  guarantee at-most-one execution during node partitions, forced deletion,
  controller races, or operator intervention; Kubernetes rollout strategy is
  not a distributed lock. A worker that requires exclusive ownership must use
  an external lease, partition assignment, or fencing token. **Scalable,
  shard-safe workers override both** `replicas` and
  `strategy: RollingUpdate` in their overlay.
- **No `ports`, no `httpGet` probes.** Liveness is "the process is running" — the
  kubelet restarts the container on exit. Adding an always-green HTTP probe to a
  process that serves no HTTP is a lie; a wrong-port probe crash-loops a healthy
  pod. Workers with a real health signal add an `exec` or `tcpSocket` probe in
  their overlay.
- **`terminationGracePeriodSeconds: 30`.** Room to drain in-flight work and
  commit on SIGTERM. Raise it for longer units of work.
- **CPU request only, no CPU limit.** Same CFS-throttling-avoidance rationale as
  the rest of the library; memory is capped.

## Compose

```yaml
resources:
  - https://gitlab.com/<org>/workload-templates-k8s//workloads/deployment-worker?ref=<tag>
  - https://gitlab.com/<org>/workload-templates-k8s//components/serviceaccount?ref=<tag>
configurations:
  - https://gitlab.com/<org>/workload-templates-k8s//kustomizeconfig.yaml?ref=<tag>
namePrefix: my-worker
labels:
  - pairs:
      app: my-worker
    includeSelectors: true
images:
  - name: PLACEHOLDER_IMAGE
    newName: registry.example.com/my-org/my-worker
    newTag: "1.0.0"
```

Then patch in the worker's specifics (env wiring, extra containers/sidecars,
volumes, securityContext, an `exec` liveness probe) in your overlay — the same
way `deployment-http` consumers do.

## Scaling out (when it is safe)

```yaml
patches:
  - target: { kind: Deployment, name: -app }
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 3
      - op: replace
        path: /spec/strategy
        value: { type: RollingUpdate, rollingUpdate: { maxUnavailable: 0, maxSurge: 1 } }
```

Only do this if concurrent instances are correct for your workload (idempotent
processing, clean partitioning, or explicitly fenced ownership).
