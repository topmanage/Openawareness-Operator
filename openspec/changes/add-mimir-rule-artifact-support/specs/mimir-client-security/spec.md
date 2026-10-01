## ADDED Requirements

### Requirement: Secret-backed Mimir authentication
The operator SHALL allow ClientConfig to reference either a bearer-token Secret or a basic-auth Secret for Mimir API authentication, and SHALL reject a configuration that selects both. Credential values SHALL not appear in ClientConfig spec/status, controller logs, Events, or client cache identifiers.

#### Scenario: Valid bearer token reference
- **WHEN** a ClientConfig references a same-namespace Secret containing a bearer token
- **THEN** Mimir requests SHALL authenticate using that token without exposing its value

#### Scenario: Conflicting authentication modes
- **WHEN** a ClientConfig configures bearer token and basic authentication together
- **THEN** the operator SHALL mark the configuration invalid and SHALL not create an authenticated Mimir client

### Requirement: Mimir TLS and timeout configuration
The operator SHALL support CA material from a ConfigMap or Secret, optional client certificate/key Secret references, and HTTP request timeout for Mimir clients. TLS peer verification SHALL be enabled by default; any insecure-skip-verify setting SHALL default to false and be explicitly documented as unsafe.

#### Scenario: Custom trusted CA
- **WHEN** a ClientConfig references a CA bundle
- **THEN** the operator SHALL use that bundle to verify the Mimir TLS endpoint

### Requirement: Secure client cache invalidation
The client cache SHALL include a fingerprint of all connection-relevant configuration, including authentication mode and referenced TLS/auth material revisions, without retaining raw secret data. The operator SHALL refresh clients when referenced Secret or ConfigMap data changes.

#### Scenario: Credential rotation
- **WHEN** a referenced authentication Secret changes
- **THEN** subsequent reconciliations SHALL use a refreshed client built from the rotated credentials
