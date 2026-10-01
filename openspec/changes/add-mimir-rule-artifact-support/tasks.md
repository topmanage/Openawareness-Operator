## 1. API and generated resources

- [ ] 1.1 Define `MimirRuleArtifact`, `RuleGroupClaim`, and status/condition types, including `adoptExisting` and validated source, checksum, refresh/cache duration, auth, and TLS constraints.
- [ ] 1.2 Extend `ClientConfig` API with mutually exclusive secret-backed authentication, TLS CA/mTLS references, timeout, and safe validation/defaulting.
- [ ] 1.3 Generate and verify deepcopy code, CRDs, RBAC manifests, sample manifests, and API documentation for both types.

## 2. Shared security and configuration foundations

- [ ] 2.1 Implement same-namespace Secret/ConfigMap reference loading with explicit key validation and redacted error helpers.
- [ ] 2.2 Implement strict `tenant-policy.yaml` ConfigMap parsing/validation, change watches and requeues, and shared authorization checks for artifact and legacy PrometheusRule write paths.
- [ ] 2.3 Extend Mimir transport/client construction for bearer/basic auth, CA/mTLS, timeout, default TLS verification, and unsafe opt-out documentation.
- [ ] 2.4 Redesign client cache identity around sanitized connection fingerprints and invalidate/rebuild clients when referenced auth/TLS resources change.

## 3. Artifact fetch, validation, and reconciliation

- [ ] 3.1 Implement a bounded, timeout-aware HTTPS artifact fetcher with validated 15s default timeout bounds, optional basic auth, custom CA, URL/header redaction, and an in-memory TTL cache.
- [ ] 3.2 Add ETag/Last-Modified conditional request handling, digest calculation and optional SHA-256 comparison, and status metadata updates.
- [ ] 3.3 Parse and validate Prometheus rule YAML with the installed rulefmt package, preserving supported group/rule fields and rejecting duplicate/conflicting group ownership.
- [ ] 3.4 Implement MimirRuleArtifact reconciliation: authorization, scheduled requeue using bounded duration defaults, exponential jittered backoff, validated group upserts, unchanged-content no-op, status Conditions, structured logs, and Events.
- [ ] 3.5 Implement hashed tuple claims and per-target locks; fail closed for claim conflicts and require explicit digest-matched adoption of unclaimed existing Mimir groups.
- [ ] 3.6 Implement managed-group tracking, update pruning limited to recorded ownership, finalizer cleanup and post-cleanup claim release limited to recorded names, and retry-safe failure handling.

## 4. Controller integration and deployment hardening

- [ ] 4.1 Register the new controller and watches for referenced Secrets/ConfigMaps and tenant policy changes without widening watch scope.
- [ ] 4.2 Apply tenant authorization to backward-compatible PrometheusRule reconciliation and document legacy annotation behavior.
- [ ] 4.3 Update manager/cache configuration and Helm values/templates for scoped watched namespaces, policy ConfigMap, response/request limits, and a pinned released image tag.
- [ ] 4.4 Reduce generated/chart RBAC to required namespaced reads and add per-namespace binding guidance.
- [ ] 4.5 Add NetworkPolicy examples for DNS and explicit Mimir/object-storage egress.

## 5. Documentation and examples

- [ ] 5.1 Add a MimirRuleArtifact sample using the grafana-resources recording-rules URL shape and secret/CA references.
- [ ] 5.2 Document artifact format, cache/refresh semantics, status/ownership lifecycle, security boundaries, and tenant-policy configuration.
- [ ] 5.3 Document durable Mimir S3-backed `ruler_storage` prerequisite and artifact group interval precedence over Mimir's default one-minute evaluation interval.

## 6. Verification

- [ ] 6.1 Add unit tests for HTTP fetch limits/timeouts, ETag/digest no-op behavior, checksum mismatch, redaction, auth, and custom TLS references.
- [ ] 6.2 Add API/client-cache tests for ClientConfig validation, TLS/auth construction, secret/config rotation, and tenant policy authorization.
- [ ] 6.3 Add envtest/controller tests for valid sync, interval preservation, unchanged ETag/digest, invalid YAML/PromQL, rejected tenant, strict policy validation, claim conflict/adoption, update pruning, finalizer cleanup scope, and credential-safe status/log/Event behavior.
- [ ] 6.4 Run gofmt, golangci-lint, unit tests, envtest/controller tests, Helm lint/template validation, and generated-manifest verification; record exact results.
