## ADDED Requirements

### Requirement: Namespace-to-tenant authorization
The operator SHALL authorize every Mimir write from MimirRuleArtifact against an allow-list mapping from Kubernetes namespace to Mimir tenants stored in the `tenant-policy.yaml` entry of a dedicated `openawareness-tenant-policy` ConfigMap in the operator namespace. The policy SHALL require `defaultDeny: true`, explicit unique namespace bindings, and nonempty unique tenant IDs; it SHALL reject wildcard, duplicate, malformed, and empty mappings. An unlisted namespace or tenant SHALL be denied before any Mimir API call, and policy changes SHALL requeue affected artifacts.

#### Scenario: Authorized artifact tenant
- **WHEN** a MimirRuleArtifact requests a tenant permitted for its Kubernetes namespace
- **THEN** the operator SHALL reconcile its validated groups to that tenant

#### Scenario: Rejected cross-tenant request
- **WHEN** a MimirRuleArtifact requests a tenant not allowed for its Kubernetes namespace
- **THEN** the operator SHALL set a clear authorization Condition and SHALL not contact Mimir

### Requirement: Legacy rule tenant protection
The operator SHALL keep PrometheusRule reconciliation backward compatible while applying the same namespace-to-tenant authorization before its Mimir mutations. Documentation SHALL identify the legacy annotation route and its authorization constraints.

#### Scenario: Legacy rule tenant denied by policy
- **WHEN** a legacy PrometheusRule resolves to a tenant not permitted for its namespace
- **THEN** the operator SHALL reject reconciliation without writing Mimir

### Requirement: Least-privilege deployment controls
The Helm deployment SHALL support configured watched namespaces and SHALL render least-privilege access to only the required resources in those namespaces plus the operator policy ConfigMap. It SHALL provide NetworkPolicy examples that permit DNS and configured Mimir/artifact endpoint egress only.

#### Scenario: Namespaced deployment rendering
- **WHEN** an administrator configures a finite watched namespace list
- **THEN** rendered RBAC and controller configuration SHALL limit watches and referenced-resource reads to that scope
