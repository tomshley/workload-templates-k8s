# Pekko cluster network isolation

Local composition of the `pekko-cluster` workload with the namespace-wide
default-deny ingress baseline and the Pekko cluster peering allow. The
`labels` transform with `includeSelectors: true` rewrites the peering
policy's `spec.podSelector` and `spec.ingress[].from[].podSelector` to the
real workload label, while the default-deny's `podSelector: {}` stays empty.
CI renders this example; the application request port is left closed for
the consumer's own allow policy.
