# Caracal Manual Intervention Analysis

## Key Patterns Discovered

### 1. APPROVAL/CONFIRMATION FLOWS (UI-Driven)
**File:** [caracal-server/.../flow/screens/provider_manager.py](caracal-server/.../flow/screens/provider_manager.py#L752)
- Line 752: `Confirm.ask("Save provider ...?", default=True)` — User must confirm provider config
- Line 913, 1057, 1263, 1325, 1659: Multiple `prompt.confirm()` calls for provider add/update/enrich
- Line 314, 393, 558, 570 in [mandate_delegation_flow.py](caracal-server/.../flow/screens/mandate_delegation_flow.py) — Mandate delegation confirmations
- Line 154-181 in [authority_ledger_flow.py](caracal-server/.../flow/screens/authority_ledger_flow.py) — Filter confirmations (UI, not auth)

**Classification:** UNNECESSARY (UI friction)
- These are UX confirmations, not security gates. Can be bypassed via non-TUI CLI/API.

### 2. ATTESTATION & PRINCIPAL LIFECYCLE
**Key Files:**
- [caracal-server/.../identity/attestation_nonce.py](caracal-server/.../identity/attestation_nonce.py#L61)
  - `issue_nonce()` (line 61) — Issues one-time nonce for attestation
  - One-time consumption semantics enforced
  - No human approval required — automated for workload identities

- [caracal-server/.../identity/principal_ttl.py](caracal-server/.../identity/principal_ttl.py#L167)
  - `register_pending_principal()` (line 167) — Registers principal in PENDING_ATTESTATION state
  - TTL-based auto-expiry if attestation not consumed
  - `_expire_pending_attestation()` (line 395) — Auto-revokes expired pending principals

- [db/models.py](caracal-server/.../db/models.py#L59)
  - `PrincipalLifecycleStatus.PENDING_ATTESTATION` — Requires attestation before ACTIVE

**Classification:** NECESSARY (but mostly automated)
- Attestation nonce is one-time, cannot be replayed
- Auto-expiry prevents indefinite pending state
- Suitable for SPIFFE/k8s workload identity attestation (signed identity → nonce consumption)

### 3. MANDATE TTL & RE-ISSUANCE PRESSURE
**File:** [caracal-server/.../core/mandate.py](caracal-server/.../core/mandate.py#L300)
- `issue_mandate()` enforces `max_validity_seconds` from AuthorityPolicy
- No automatic renewal mechanism found
- Short TTLs force manual re-issuance by human or orchestrator

**Authority Policy Constraints:**
- `max_validity_seconds`: Per-principal cap on mandate lifetime
- Flow screen: [authority_policy_flow.py](caracal-server/.../flow/screens/authority_policy_flow.py) requires manual policy creation

**Observation:**
- Orchestrators with `allow_delegation=true` can re-issue to subordinates within their TTL
- No auto-renewal or refresh-token pattern detected

**Classification:** UNNECESSARY (can be automated)
- Scoped child mandates can extend orchestrator authority at runtime without human touch
- TTL caps prevent indefinite grants, still enforced

### 4. ENTERPRISE ADMIN BOOTSTRAP
**File:** [caracalEnterprise/services/vault/app.py](caracalEnterprise/services/vault/app.py#L333)
- `admin_bootstrap()` (line 333) — Vault bootstraps admin keypair
- `bootstrap_keypair()` (line 436) — Generate keys for principal

**No human approval observed** — direct API call based on vault token auth

**Observation from [caracal-architecture.md](/memories/repo/caracal-architecture.md#L26):**
- "Enterprise admin bootstrap identities must be linked by stable enterprise user metadata before registering by name"
- Prevents re-registration issues via idempotent linking

**Classification:** NECESSARY (but well-designed)
- No manual hand-off, token-gated
- Idempotent by principal metadata linking

### 5. PROVIDER CREDENTIAL ENTRY
**File:** [caracal-server/.../flow/screens/provider_manager.py](caracal-server/.../flow/screens/provider_manager.py#L1180-1325)
- Line 1266-1325: `_collect_connection_settings()` — Human must paste API key or select "none"
- Line 1029: `store_workspace_provider_credential()` — Stores to vault
- Line 750: Status displays "configured" or "missing"

**Flow:**
1. Human enters provider name
2. Selects auth scheme (api_key, header, oauth2, etc.)
3. Manually pastes credential value into TUI
4. Stored encrypted to vault

**Classification:** UNNECESSARY (can be improved)
- Could pull from environment variables, Vault directly, or OAuth2 PKCE flow
- No security gate: vault encryption already protects at rest

### 6. REVOCATION SYSTEM
**File:** [caracal-server/.../core/mandate.py](caracal-server/.../core/mandate.py#L561)
- `revoke_mandate()` requires issuer/principal with revoke authority
- No human approval required

**File:** [caracal-server/.../core/revocation.py](caracal-server/.../core/revocation.py)
- Cascade revocation: leaves-first topological ordering
- Automated revocation on principal lifecycle transition

**Classification:** MOSTLY AUTOMATED (some UI points)
- Revocation itself is automatic on lifecycle change
- Manual revocation via TUI at [mandate_delegation_flow.py#L558](caracal-server/.../flow/screens/mandate_delegation_flow.py#L558): `confirm("Revoke downstream edges?", default=True)`
- This confirmation is UNNECESSARY (UX friction, not security gate)

### 7. DELEGATION AUTHORITY
**File:** [caracal-architecture.md](/memories/repo/caracal-architecture.md#L56-L71)
- Allowed: human → orchestrator, orchestrator → worker, worker → service
- `allow_delegation=true` flag on AuthorityPolicy
- `max_network_distance` limits hops

**Observation:**
- One human-issued root mandate → orchestrator can create unbounded child delegations (within distance limit)
- No re-approval needed for each child delegation

**Classification:** WELL-DESIGNED
- Proper autonomy model; human sets policy once

### 8. SESSION MANAGEMENT & LEDGER INSPECTION
**File:** [caracal-server/.../identity/ais_server.py](caracal-server/.../identity/ais_server.py)
- Session tokens issued automatically
- Authority ledger events recorded immutably
- No human inspection required for normal operation

**Classification:** AUTOMATED (inspection for audit, not operation)

## IDEMPOTENCY GAPS

### Principal Re-registration
**Issue:** [principal_flow.py#L165-179](caracal-server/.../flow/screens/principal_flow.py#L165-179)
- No check for existing principal by name before creation
- Flow requires manual duplicate detection

**Mitigation Found:** [caracal-architecture.md#L26](/memories/repo/caracal-architecture.md#L26)
- Enterprise uses principal metadata linking, not name-based identity
- Prevents issues when bootstrap is retried

**Status:** PARTIALLY ADDRESSED
- OSS flow lacks idempotency; enterprise has it via metadata linking

### Provider Registration
**File:** [provider_manager.py#L752](caracal-server/.../flow/screens/provider_manager.py#L752)
- Allows overwriting existing provider by name
- No dry-run or diff preview

**Status:** FUNCTIONAL (manual overwrite accepted)

## KEY AUTOMATION OPPORTUNITIES

1. **Auto-renewal of child mandates** — Orchestrator reissues to workers with same scopes, reset TTL, no human touch
2. **Provider credential from env/vault** — Eliminate manual paste
3. **Principal idempotency** — Check existing by metadata before create
4. **UI confirmations → API flags** — Non-interactive automation modes
5. **Attestation auto-transition** — Once nonce consumed, auto-promote to ACTIVE
6. **Workspace idempotency** — Repeated creation should be no-op

## SECURITY GATES THAT MUST REMAIN MANUAL

1. **Issuer Authority Policy** — Human defines max_validity_seconds, allowed_actions, allowed_resource_patterns
2. **Delegation Direction Rules** — Principal kind constraints (human→orch, orch→worker, no →human)
3. **Attestation Requirement** — Orchestrator/worker must have attestation nonce consumed
4. **Principal Lifecycle Enforcement** — Cannot issue mandate to PENDING_ATTESTATION principal
5. **Revocation on Lifecycle** — Auto-revoke leaves-first on principal state change

## ANTI-PATTERNS

None found. Caracal architecture is clean:
- No approval queues that block execution
- No redundant human-in-loop for delegated authorities
- Proper separation of policy definition (human) from policy enforcement (automatic)
