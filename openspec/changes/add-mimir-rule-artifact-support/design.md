## Context

The operator currently reconciles inline rule definitions through `PrometheusRule` and `LokiRuleGroup`, and `MimirAlertTenant` reads template values only from same-namespace Secrets and ConfigMaps. Mimir clients are built from an address-only `ClientConfig`; cache identity does not capture authentication, TLS trust, certificates, or referenced-resource revisions. The new artifact source must integrate without allowing the object storage URL or a free-form annotation to bypass tenant boundaries.

The design adds a namespaced `MimirRuleArtifact` dedicated to HTTP(S) Prometheus rule artifacts, while preserving existing `PrometheusRule` support as legacy-compatible behavior.

## Goals / Non-Goals

**Goals:**
- Reconcile standard Prometheus `groups` YAML from HTTP(S), including S3-compatible presigned or gateway URLs.
- Make fetches bounded, conditional, observable, integrity-checked, and safe to retry.
- Safely own only the Mimir groups created by each artifact CR across updates and deletion.
- Secure Mimir and artifact requests with Kubernetes-reference-backed authentication and TLS material.
- Enforce operator-configured namespace-to-tenant authorization before any Mimir write.

**Non-Goals:**
- Implement native S3 protocol/listing, write artifacts to object storage, or support arbitrary non-HTTP schemes.
- Import or manage Alertmanager configuration through this CR.
- Adopt existing unowned Mimir groups, reconcile individual rules independently, or remove rule groups owned by another CR.
- Guarantee immediate refresh on remote object change; periodic reconciliation is the delivery contract.

## Decisions

### Dedicated artifact CR and data model

`MimirRuleArtifact` is namespaced and contains `clientRef.name`, `tenant`, `rulerNamespace`, `url`, `refreshInterval`, `contentCacheDuration`, optional SHA-256 checksum, optional basic-auth Secret key references, optional TLS CA ConfigMap reference, and `adoptExisting`. Defaults and validation are webhook/schema-backed where possible: HTTPS is required; `refreshInterval` defaults to 5m and is bounded to 1m–1h; `contentCacheDuration` defaults to 1m and is bounded to 0s–5m and no greater than refresh; and insecure TLS is false by default.

Status stores `observedGeneration`, sanitized source metadata (`etag`, `lastModified`, content SHA-256), `managedGroupNames`, `lastSuccessfulFetchTime`, `lastSyncTime`, and Conditions. It never stores URLs, credentials, bodies, Authorization headers, or secret contents. Storing group names makes delete/update ownership deterministic across restart and avoids listing-and-pruning unrelated Mimir state.

Alternatives considered: extending `PrometheusRule` conflates Kubernetes-authored and remote-authoritative configuration; a generic source CR adds indirection before a demonstrated second consumer exists.

### Fetch and cache strategy

Use a controller-owned HTTP client with a request timeout and a hard `io.LimitReader` response limit. The reconciler persists validator metadata in status and sends `If-None-Match` and/or `If-Modified-Since` on each refresh after the cache TTL expires. A 304 response reuses the last known successful digest and skips parse/upload. An in-process, bounded TTL cache holds the validated bytes keyed by CR UID plus URL and validator state, avoiding status body storage. On cache expiry/restart, a conditional fetch remains correct.

Successful 200 content is SHA-256 hashed, optionally checked against the supplied lowercase/uppercase hex digest using constant-time comparison, then parsed using the installed Prometheus `rulefmt` parser/validator. Only a successful validation can mutate Mimir. Content whose digest matches the last successfully reconciled digest skips Mimir upserts. HTTP failures retain the last good Mimir state, set a failure condition, emit a sanitized Event/log, and requeue using exponential jittered backoff from 30s capped at 15m.

Alternatives considered: only content hashing (unnecessarily downloads every interval) and storing content in status/ConfigMaps (leaks sensitive configuration and inflates the API server).

### Managed group reconciliation

After tenant authorization and artifact validation, the controller calls the existing Mimir client per desired `rulefmt.RuleGroup`; the dependency DTO is used directly to preserve its supported fields (including interval, limit, labels, annotations, `for`, and `keep_firing_for`). It computes `previousManaged - desired` from status and deletes exactly that set. It never lists all backend groups to infer ownership. A finalizer repeats deletion of exactly `status.managedGroupNames`, then removes itself; failures retain the finalizer for retry.

Because Mimir identifies groups only by tenant, ruler namespace, and group name, the tuple is globally exclusive. Before a push, the reconciler creates or renews a cluster-side `RuleGroupClaim` (Lease-like registry) keyed by a stable hash of that tuple and takes a per-target reconciliation lock. A conflicting claim fails closed. If Mimir already has a group without a claim, reconciliation rejects it by default; adoption requires `spec.adoptExisting: true` and a digest match against the desired group. Mimir existence is never evidence of ownership. Claims are released only after successful managed-group cleanup.

### Connection security and client cache

`ClientConfig` gains a mutually exclusive bearer-token Secret reference or basic-auth Secret reference, CA ConfigMap/Secret reference, optional mTLS cert/key Secret reference, and request timeout. Reference objects are same namespace as the `ClientConfig`; required keys are explicit. TLS verification stays enabled unless an explicitly documented unsafe opt-out is configured.

Build a normalized connection fingerprint from address, timeout, TLS flags/material digests, and auth mode/material digests—not raw secret values. The cache key includes this fingerprint. Controllers watch referenced Secrets/ConfigMaps and invalidate/reconcile dependent clients when their resource version/data changes. No credential or URL query appears in status, Events, or structured logs.

### Tenant authorization, watches, and RBAC

An operator-namespace `openawareness-tenant-policy` ConfigMap contains `tenant-policy.yaml`: a versioned TenantPolicy document with required `defaultDeny: true` and explicit bindings from unique Kubernetes namespaces to nonempty, unique tenant lists. Wildcards, empty lists, duplicate namespaces/tenant IDs, malformed identifiers, and non-default-deny policy are invalid. A shared authorizer checks `MimirRuleArtifact.spec.tenant` before client creation or Mimir calls. A rejected request writes an explicit authorization Condition, emits a sanitized Warning Event, and does not contact Mimir. Policy updates requeue affected artifacts. Legacy PrometheusRule routes through the same authorizer, resolving its tenant using the existing backward-compatible inputs only where policy permits; documentation marks annotations as legacy.

Manager cache/watch namespaces are supplied by Helm values. RBAC markers and rendered roles grant namespaced `get/list/watch` only for the CRs and same-namespace Secrets/ConfigMaps actually consumed; the policy ConfigMap is readable in the operator namespace. Deployments needing multiple watched namespaces use per-namespace RoleBindings rather than blanket cluster-wide read.

### Operational interfaces

The controller emits Normal/Warning Events for fetch, validation, authorization, sync, and cleanup transitions, using object name/namespace and redacted source identity only. Helm exposes the response-size limit, artifact timeout, watched namespaces, policy ConfigMap name, and image tag. NetworkPolicy samples allow DNS plus explicit Mimir and artifact-storage destinations. Documentation explains the required durable Mimir `ruler_storage` (such as S3) and rule group interval precedence over Mimir's 1m default.

## Risks / Trade-offs

- [Presigned URLs can contain secrets] → Never persist or log full URLs; redact query strings and authorization-related headers in errors and Events.
- [Status-based ownership can be lost through manual status mutation] → Protect status RBAC, only delete names in status, and fail closed on malformed ownership state.
- [Mimir API has no portable ownership metadata] → Enforce exclusive tuple ownership with a claim registry and per-target lock; reject unclaimed pre-existing groups except explicit digest-matched adoption.
- [Artifact endpoint outages] → Retain last good remote state, report condition/event, use bounded retries, and avoid deletion on fetch failure.
- [Secret rotation and cache staleness] → Fingerprint material revisions and watch references to invalidate clients; bound cache lifetime.
- [Cross-namespace policy complexity] → Provide Helm-managed per-namespace RBAC and strict explicit policy validation; reject unconfigured namespaces by default.

## Migration Plan

1. Ship CRD/controller and disabled-by-default Helm feature flags with no behavior change for existing resources.
2. Configure watched namespaces, tenant-policy ConfigMap, Mimir `ClientConfig` credentials/TLS, RBAC bindings, and NetworkPolicy egress.
3. Confirm Mimir uses durable `ruler_storage` before creating artifacts.
4. Apply `MimirRuleArtifact` resources alongside legacy rules; verify status, Events, and Mimir groups.
5. Migrate producers to artifact URLs and retire inline legacy rules only after ownership and backend behavior are verified.
6. Roll back by deleting artifact CRs (finalizer removes only their recorded groups) or disabling reconciliation after preserving desired state according to the documented operational procedure.

## Open Questions

- What Kubernetes API and RBAC scope best fits the `RuleGroupClaim` registry: a dedicated namespaced CRD, Lease, or controller-managed ConfigMap entry?
