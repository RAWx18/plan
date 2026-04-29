# Autonomous AI Trust & Security Research Brief
*Research conducted 2026-04-29*

## Key Codebase Components Identified

### Caracal (OSS + Enterprise)
- **Location**: /caracal (core) + /caracalEnterprise (enterprise)
- **Primary Pattern**: Pre-execution authorization gating with explicit mandate model

### Core Systems Found

#### 1. Audit & Tamper-Evident Logs
- [caracal/core/audit.py](caracal/packages/caracal-server/caracal/core/audit.py) — Enhanced audit references with:
  - SHA-256 & SHA3-256 hash algorithm support (crypto agility)
  - Chain verification (previous_hash for blockchain-style integrity)
  - ECDSA P-256 signatures for non-repudiation
  - Tamper detection via hash comparison
- [caracalEnterprise/tamper_detection.py](caracalEnterprise/services/api/src/caracal_api/services/tamper_detection.py) — Anomaly detection:
  - Sequence gap detection (missing events)
  - HMAC signature failure detection
  - Heartbeat lapse monitoring
  - Usage anomaly detection
  - Severity classification (WARNING, CRITICAL, FRAUD_SUSPECTED)

#### 2. Policy-as-Code & Authorization
- [caracal/db/models.py](caracal/packages/caracal-server/caracal/db/models.py) — AuthorityPolicy model:
  - Resource patterns (regex/glob support)
  - Action scopes
  - Delegation constraints with network distance
  - TTL-based validity periods
  - Active/inactive soft-delete pattern
- [caracalEnterprise/routes/policies.py](caracalEnterprise/services/api/src/caracal_api/routes/policies.py) — Policy management API:
  - Monotonic policy restriction (tighten-only operations)
  - Pattern validation & scope enforcement
  - Authority checking via mandates

#### 3. Cryptographic Signing & Verification
- [caracal/merkle/signer.py](caracal/packages/caracal-server/caracal/merkle/signer.py) — Software signer:
  - ECDSA P-256 signing (RFC 6979 deterministic)
  - PEM key storage with passphrase encryption
  - Key rotation support
- [caracal/merkle/verifier.py](caracal/packages/caracal-server/caracal/merkle/verifier.py) — Batch verification:
  - Merkle tree integrity verification
  - Signature validation
  - Migration batch handling with reduced guarantees

#### 4. Human-in-the-Loop & Workflows
- [caracalEnterprise/models/workflow.py](caracalEnterprise/services/api/src/caracal_api/models/workflow.py) — Workflow automation:
  - Step-based execution (issue_mandate, revoke_mandate, delegate_authority, register_principal)
  - Execution context & results tracking
  - Error handling with step-level granularity
- [caracalEnterprise/routes/workflows.py](caracalEnterprise/services/api/src/caracal_api/routes/workflows.py) — Workflow API:
  - License gating for enterprise features
  - Permission-based access control (write:workflows, read:workflows)
  - Execution tracking

#### 5. Least-Privilege & Access Control
- [caracal/core/allowlist.py](caracal/packages/caracal-server/caracal/core/allowlist.py) — AllowlistManager:
  - LRU cache (1000 pattern cache, 500 principal cache)
  - Regex & glob pattern matching
  - Per-principal resource allowlists
  - p99 latency target < 2ms
- [caracalEnterprise/middleware/authority.py](caracalEnterprise/services/api/src/caracal_api/middleware/authority.py) — Authority middleware:
  - Pre-route authorization checks
  - Mandate validation via Caracal Core client
  - Delegation suggestion & guidance

#### 6. Agent Registration & Identity
- Principals (named agents/users) with:
  - Unique principal_id (UUID)
  - Policy constraints per principal
  - Delegation network distance controls
- Mandates (delegation tokens) with:
  - Resource scope binding
  - Action scope binding
  - TTL-based validity
  - Revocation tracking

#### 7. Incident Response
- [caracal/INCIDENT_RESPONSE.md](caracal/INCIDENT_RESPONSE.md) — Formal IRP:
  - Red/Blue team structure
  - Type A-D classification
  - 7-day initial review target
  - Containment → Mitigation → Analysis → Fix cycle
  - Two-layer approach (immediate guard + structural fix)

## Industry Standards Referenced

- **NIST Zero Trust Architecture (SP 800-207)**: Focus on pre-execution verification, deny-by-default, and explicit validation
- **Threat Model**: Detailed in [caracal/THREAT_MODEL.md](caracal/THREAT_MODEL.md)
  - Trust boundaries clearly defined
  - Fail-closed behavior mandated
  - Audit trail as first-class requirement

## Concepts Modeled

Core concepts from documentation:
- Authority Enforcement Model (pre-execution gating)
- Mandate (delegation token with scope)
- Principal (named agent/user)
- Policy (resource patterns, action scopes, validity TTL)
- Workspace (isolation boundary)
- Ledger (immutable event log)
- Caveat (constraint on delegation)
- Delegation (network distance, depth control)

## Enterprise-Specific Patterns

- Centralized authority control from dashboard
- Gateway enforcement of provider traffic
- Enterprise sync for workspace management
- SSO/RBAC for multi-tenant access
- Compliance-focused audit trails
- Tamper detection with severity classification
- Workflow automation with license gating
