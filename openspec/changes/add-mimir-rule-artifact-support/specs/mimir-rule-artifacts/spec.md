## ADDED Requirements

### Requirement: Artifact resource source and scheduling
The operator SHALL provide a namespaced `MimirRuleArtifact` resource with a `clientRef`, tenant, ruler namespace, HTTPS URL, refresh interval, cache duration, optional SHA-256 checksum, optional basic-auth Secret reference, optional TLS CA ConfigMap reference, and optional explicit adoption control. `refreshInterval` SHALL default to 5m and be limited to 1m–1h; `contentCacheDuration` SHALL default to 1m, be limited to 0s–5m, and not exceed refresh interval. It SHALL reconcile the resource no less often than its configured refresh interval and SHALL enforce a 15s default request timeout bounded to 1s–60s and a bounded response size.

#### Scenario: Periodic HTTPS artifact fetch
- **WHEN** a valid artifact resource reaches its refresh interval
- **THEN** the operator fetches its HTTPS artifact within configured timeout and size limits

#### Scenario: Oversized response
- **WHEN** an artifact response exceeds the configured maximum size
- **THEN** the operator SHALL reject it without parsing or calling Mimir and report a failure Condition

### Requirement: Conditional and unchanged artifact reconciliation
The operator SHALL use stored ETag and Last-Modified validators in conditional requests when supplied by the artifact endpoint. It SHALL skip Mimir mutations for a not-modified response or a successfully fetched artifact whose digest equals the last reconciled digest.

#### Scenario: ETag returns not modified
- **WHEN** the endpoint responds 304 to an If-None-Match request
- **THEN** the operator SHALL retain the current managed groups and record a successful refresh without uploading groups

#### Scenario: Content digest is unchanged
- **WHEN** a fetched artifact has the same SHA-256 digest as the last reconciled content
- **THEN** the operator SHALL not upsert or delete Mimir rule groups

### Requirement: Artifact validation and fidelity
The operator SHALL verify an optional SHA-256 checksum before parsing and SHALL parse and validate standard Prometheus rule YAML using the installed Prometheus rulefmt validation before any Mimir API mutation. It SHALL pass validated rule groups to Mimir without discarding fields supported by that dependency.

#### Scenario: Valid recording rule with interval
- **WHEN** an artifact contains a valid group with `interval`, labels, annotations, and supported rule fields
- **THEN** the operator SHALL reconcile the same group definition to the selected Mimir ruler namespace and preserve those fields

#### Scenario: Invalid checksum or rule content
- **WHEN** checksum verification, YAML parsing, or rulefmt validation fails
- **THEN** the operator SHALL not call Mimir, retain the last good managed state, and set a validation failure Condition

### Requirement: Exclusive managed group ownership
The operator SHALL treat `(tenant, rulerNamespace, groupName)` as globally exclusive and SHALL establish a cluster-side hashed `RuleGroupClaim` registry entry and per-target reconciliation lock before writing a group to Mimir. It SHALL fail closed when another artifact owns the claim. A group found in Mimir with no matching claim SHALL be rejected unless `spec.adoptExisting` is true and its definition matches the desired group digest; Mimir existence alone SHALL NOT establish ownership.

#### Scenario: Group claimed by another artifact
- **WHEN** an artifact contains a group with a target tuple claimed by another artifact
- **THEN** the operator SHALL reject reconciliation for that group without changing Mimir

#### Scenario: Explicit matching adoption
- **WHEN** `spec.adoptExisting` is true and a pre-existing unclaimed Mimir group matches the desired group digest
- **THEN** the operator SHALL create its claim and record the group as managed

### Requirement: Managed group lifecycle
The operator SHALL record the names of groups it successfully manages for each artifact resource. On a valid update it SHALL remove only names previously recorded for that resource that are absent from the desired artifact. On deletion it SHALL use a finalizer to delete only the recorded managed groups, release their claims only after successful cleanup, and retain the finalizer until cleanup succeeds.

#### Scenario: Artifact removes a group
- **WHEN** an updated valid artifact omits a group previously recorded as managed
- **THEN** the operator SHALL delete that group from the resource's Mimir tenant and ruler namespace without deleting unrelated groups

#### Scenario: Artifact resource deletion
- **WHEN** a MimirRuleArtifact is deleted after managing groups
- **THEN** its finalizer SHALL delete only its recorded groups before allowing Kubernetes deletion

### Requirement: Artifact status and safe observability
The operator SHALL persist observed generation, managed group names, artifact digest, available ETag/Last-Modified metadata, successful sync timestamps, and Conditions. Logs and Kubernetes Events SHALL omit credentials, Authorization headers, secret values, artifact content, and URL query strings.

#### Scenario: Artifact fetch fails
- **WHEN** an artifact request fails
- **THEN** the operator SHALL emit a sanitized Warning Event and structured log, set a failure Condition, and leave previously reconciled Mimir groups unchanged
