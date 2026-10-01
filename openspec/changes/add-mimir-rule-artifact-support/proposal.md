## Why

Recording rules authored outside the cluster must currently be copied into Kubernetes resources before this operator can reconcile them to Mimir. This creates a manual, error-prone deployment path and prevents grafana-tmcloud-produced rule artifacts from being used as a versioned source of truth. The operator needs a secure, tenant-isolated way to fetch and reconcile those artifacts directly.

## What Changes

- Add a namespaced `MimirRuleArtifact` CRD that fetches Prometheus rule YAML from HTTPS object-storage or HTTP artifact endpoints and reconciles its rule groups into a selected Mimir tenant and ruler namespace.
- Validate artifact integrity, size, YAML, and PromQL/rule semantics before performing Mimir mutations; preserve supported Prometheus rule-group and rule fields.
- Add conditional HTTP fetching, bounded in-memory content caching, refresh scheduling, status tracking, and managed-group-only cleanup through a finalizer.
- Extend `ClientConfig` with secret-backed Mimir authentication, TLS trust/client-certificate configuration, and request timeout controls.
- Enforce namespace-to-Mimir-tenant allow-list policy sourced from an operator-namespace ConfigMap; apply the same protection path to legacy `PrometheusRule` reconciliation while keeping that feature backward compatible.
- Reduce RBAC to namespaced least privilege, add namespace watch scoping configuration, NetworkPolicy examples, Helm configuration, samples, and operational documentation.
- Pin Helm image defaults to a released version rather than `latest`.

## Capabilities

### New Capabilities
- `mimir-rule-artifacts`: Securely fetch, validate, reconcile, refresh, and safely clean up externally hosted Prometheus rule artifacts.
- `mimir-client-security`: Configure authenticated, TLS-secured Mimir clients from Kubernetes references without exposing credential material.
- `mimir-tenant-authorization`: Authorize Mimir writes against namespace-to-tenant policy for artifact and legacy rule resources.

### Modified Capabilities

- None; no baseline OpenSpec capabilities exist in this repository.

## Impact

- New `openawareness.syndlex/v1beta1` API type, generated CRD, controller, status/conditions, finalizer, and tests.
- Changes to Mimir client creation and cache invalidation, `ClientConfig`, legacy PrometheusRule reconciliation, RBAC markers/manifests, manager configuration, and Helm chart values/templates.
- Adds secure HTTP/TLS, Prometheus rule validation, Kubernetes Event emission, and object/artifact cache behavior.
- Requires Mimir deployments to use durable `ruler_storage` (for example S3) for API-managed rules; ConfigMap-mounted local rule files are unsuitable.
