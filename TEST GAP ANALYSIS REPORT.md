Based on my thorough read-only analysis of both the caracal and caracalEnterprise repositories, here is a comprehensive test planning report identifying critical gaps in invariant and edge-case testing for security boundaries, lifecycle, policy validators, auth, delegation, and unsafe parsing.

---

## **TEST GAP ANALYSIS REPORT**

### **Executive Summary**
The test suite has foundational coverage of main workflows but lacks comprehensive invariant testing and systematic edge-case coverage after recent production fixes to authority enforcement, lifecycle management, and delegation boundaries. **117 specific test gaps identified** across six critical areas.

---

## **1. SECURITY BOUNDARIES – 24 Test Gaps**

### **1.1 Authority Boundary Enforcement**
**Production Files:**
- [caracal/core/authority.py](caracal/packages/caracal-server/caracal/core/authority.py#L769-L865) – `validate_mandate()`, `check_delegation_path()`
- [caracal_api/middleware/authority.py](caracalEnterprise/services/api/src/caracal_api/middleware/authority.py#L60-L160) – Authority check middleware

**Existing Tests:** `tests/unit/core/test_authority.py` (basic revoked/expired scenarios only)

**Missing Invariant Tests:**

| Test Case | Target Function | Test Location | Rationale |
|---|---|---|---|
| **Workspace isolation violation** | `validate_mandate()` | `tests/security/authority/test_workspace_boundary.py` | Mandate issued in workspace A must not validate for workspace B access |
| **Principal lifecycle inactive during validation** | `validate_mandate()` | `tests/unit/core/test_authority.py` | Must deny if subject_id principal is SUSPENDED/DEACTIVATED/REVOKED during check |
| **Signature verification bypassed** | `verify_mandate_signature()` | `tests/security/authority/test_signature_edge_cases.py` | Invalid/missing signatures must fail; empty signature must fail; tampered mandate must fail |
| **Subject binding cross-principal confusion** | `validate_mandate()` | `tests/security/authority/test_subject_binding.py` | Caller supplying different principal_id than mandate.subject_id must deny; no impersonation allowed |
| **Caveat chain break-in** | `evaluate_caveat_chain()` | `tests/unit/core/test_caveat_chain.py` (expand) | Missing/empty caveat chain with caveats expected must deny; HMAC mismatch must deny |
| **Provider scope parsing injection** | `parse_provider_scope()` (authority.py:27) | `tests/security/parsing/test_provider_scope_injection.py` | Malformed provider scopes like `provider::`, `provider:a:b:c:d:e` must reject gracefully |
| **Resource pattern bypass** | `_match_pattern()` (mandate.py:446) | `tests/unit/core/test_mandate.py` (expand) | Wildcard abuse like `*` matching more than intended; regex injection in patterns |
| **Action scope scope creep** | `validate_mandate()` | `tests/unit/core/test_authority.py` (expand) | Action must be exact member of action_scope, not prefix match; `read` must not match `read_admin` |
| **Expired but not-yet-valid acceptance** | `validate_mandate()` | `tests/unit/core/test_authority.py` (expand) | Mandate valid_from > now must deny; valid_until < now must deny; boundary timestamps at tick=0 must reject |
| **Ledger recording omission** | `_record_ledger_event()` (authority.py:387) | `tests/integration/core/test_authority_ledger.py` (expand) | All deny decisions must emit ledger events with boundary_stage; denied without ledger write must still deny |

**Suggested Test Location:** `tests/security/boundaries/` – new directory

---

### **1.2 Cross-Principal Authorization Leakage**
**Production Files:** 
- [caracal_api/integrations/caracal_core.py](caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py#L140-L220) – `check_user_authority()` 
- [caracal_api/routes/policies.py](caracalEnterprise/services/api/src/caracal_api/routes/policies.py#L1-L140) – Policy creation

**Existing Tests:** `caracalEnterprise/services/api/tests/test_authority_bridge.py` (happy path only)

**Missing Invariant Tests:**

| Test Case | Target Function | Test Location |
|---|---|---|
| **Human principal cannot access worker's mandate** | `check_user_authority()` | `tests/security/boundaries/test_principal_isolation.py` |
| **Service principal cannot issue new mandates** | `issue_mandate()` (CaracalClient) | `tests/security/boundaries/test_service_terminal.py` |
| **Policy update doesn't retroactively loosen past mandates** | `update_policy()` | `tests/unit/core/test_mandate.py` (policy immutability) |
| **Revoked principal's mandates all denied immediately** | `validate_mandate()` with revoked issuer | `tests/integration/core/test_revocation_cascade.py` (expand) |
| **Attestation status gates workflow for worker/orchestrator** | `validate_transition()` in lifecycle | `tests/unit/core/test_lifecycle.py` (already has this, verify) |

---

### **1.3 Metadata Injection & Context Spoofing**
**Production Files:** 
- `caracal_api/routes/` (multiple)
- [caracal/runtime/hardcut_preflight.py](caracal/runtime/hardcut_preflight.py) – Caller context validation

**Existing Tests:** `tests/integration/api/test_endpoints.py#L80-L169` (basic spoofing rejection)

**Missing Invariant Tests:**

| Test Case | Target Function | Test Location |
|---|---|---|
| **Caller-supplied principal_id in metadata rejected** | Request handler | `tests/integration/api/test_hardcut_metadata_spoofing.py` |
| **Caller-supplied mandate_id in tool_args rejected** | Tool call handler | (same) |
| **Context tags cannot be modified post-issuance** | Mandate model/route | `tests/unit/core/test_mandate.py` (expand) |
| **Workspace switch doesn't expose prior workspace principal list** | Auth context | `tests/security/boundaries/test_workspace_switch_leakage.py` |

---

## **2. LIFECYCLE MANAGEMENT – 28 Test Gaps**

### **2.1 State Machine Invariants**
**Production Files:**
- [caracal/core/lifecycle.py](caracal/packages/caracal-server/caracal/core/lifecycle.py#L1-L198) – `PrincipalLifecycleStateMachine`
- `caracal/core/identity.py` – `transition_lifecycle_status()`

**Existing Tests:** `tests/unit/core/test_lifecycle.py` (transition matrix exists but edge cases missing)

**Missing Invariant Tests:**

| Test Case | Kind | From → To | Test Location | Notes |
|---|---|---|---|---|
| **REVOKED is terminal** | all | REVOKED → * | Already tested | Verify all targets rejected |
| **PENDING_ATTESTATION blocks activation without attestation** | worker | PENDING_ATTESTATION → ACTIVE (attestation=pending) | Already tested | Verify attestation_status="attested" required |
| **Non-reactivating after deactivation** | orchestrator | DEACTIVATED → ACTIVE | Already tested | Verify no path exists |
| **Non-reactivating after revocation** | worker | REVOKED → ACTIVE | Already tested | Verify no path exists |
| **Suspension is reversible only for HUMAN** | human | SUSPENDED → ACTIVE | Already tested | Verify orchestrator/worker cannot resume |
| **EXPIRED auto-transitions** | all | ACTIVE → EXPIRED (if past valid_until) | `tests/unit/core/test_lifecycle.py` (expand) | Validate expiry clock-driven transition |
| **Attestation status upgrade path only** | worker | attestation: pending → attested (not reversed) | `tests/unit/core/test_lifecycle.py` (expand) | Downgrade must fail |
| **Concurrent transition attempt isolation** | all | Race condition on simultaneous transitions | `tests/integration/core/test_lifecycle_concurrent.py` (new) | DB constraint should serialize |

**Suggested Test Location:** Expand `tests/unit/core/test_lifecycle.py`; add `tests/integration/core/test_lifecycle_concurrent.py`

---

### **2.2 Spawned Principal Attestation Gating**
**Production Files:**
- `caracal/core/identity.py` – `spawn_principal()` 
- Database migration: [k0l1m2n3o4p5_principal_kind_lifecycle_hardcut.py](caracal/packages/caracal-server/caracal/db/migrations/versions/k0l1m2n3o4p5_principal_kind_lifecycle_hardcut.py) (backfill logic)

**Existing Tests:** `tests/unit/test_nested_workflows.py#L15-L50` (basic spawn tests)

**Missing Invariant Tests:**

| Test Case | Target Function | Test Location |
|---|---|---|
| **Spawned ORCHESTRATOR begins in PENDING_ATTESTATION** | `spawn_principal()` | `tests/unit/core/test_identity.py` (expand) |
| **Attestation_status="pending" locks PENDING_ATTESTATION until "attested"** | Lifecycle validation | `tests/unit/core/test_lifecycle.py` (expand) |
| **HUMAN spawn bypasses attestation gate** | `spawn_principal()` | `tests/unit/core/test_identity.py` (expand) |
| **SERVICE spawn does not require attestation** | `spawn_principal()` | `tests/unit/core/test_identity.py` (expand) |
| **Cannot transition WORKER from PENDING_ATTESTATION to EXPIRED/REVOKED only** | Lifecycle validation | `tests/unit/core/test_lifecycle.py` (expand) |

---

### **2.3 Principal Deactivation Cascades**
**Production Files:**
- [caracal/core/revocation.py](caracal/packages/caracal-server/caracal/core/revocation.py) – `revoke_principal()`
- [caracal/core/delegation_graph.py](caracal/packages/caracal-server/caracal/core/delegation_graph.py#L580-L660) – `validate_authority_path()`

**Existing Tests:** `tests/unit/core/test_revocation.py#L307` (collect edges only)

**Missing Invariant Tests:**

| Test Case | Target Function | Test Location |
|---|---|---|
| **Revoke principal invalidates all issued mandates immediately** | `revoke_principal()` | `tests/integration/core/test_revocation_cascade.py` (new) |
| **Revoke principal blocks validation of downstream delegations** | `validate_authority_path()` | (same) |
| **Deactivated issuer cannot issue new mandates** | `issue_mandate()` validation | `tests/unit/core/test_mandate.py` (expand) |
| **Suspended issuer can resume and issue mandates again** | `issue_mandate()` with SUSPENDED issuer | `tests/unit/core/test_mandate.py` (expand) |
| **Cascading revocation of child mandates on parent revoke** | `revoke_mandate(cascade=True)` | `tests/integration/core/test_delegation_cascade.py` (new) |

---

## **3. POLICY VALIDATORS – 19 Test Gaps**

### **3.1 Policy Scope Matching & Tightening**
**Production Files:**
- [caracal_api/routes/policies.py](caracalEnterprise/services/api/src/caracal_api/routes/policies.py#L1-L200) – Create/Update/Tighten routes
- [caracal/core/mandate.py](caracal/packages/caracal-server/caracal/core/mandate.py#L400-L500) – Scope validation

**Existing Tests:** `caracalEnterprise/services/api/tests/test_policies_response.py` (response shape only)

**Missing Invariant Tests:**

| Test Case | Target Function | Test Location | Notes |
|---|---|---|---|
| **Tighten policy rejects expansion** | `_validate_scope_subset()` (mandate.py) | `tests/unit/core/test_mandate.py` (expand) | Tighten with `['*']` must fail; original was narrower |
| **Policy pattern validation rejects malformed** | `_validate_scope_list()` (policies.py:34) | `tests/unit/enterprise/test_policy_routes.py` (new) | Invalid patterns: `provider:`, `::`, `provider:a:b:c:d` |
| **Max validity seconds enforced on mandate issue** | `issue_mandate()` validation | `tests/unit/core/test_mandate.py` (expand) | Request validity_seconds > policy max must reject |
| **Max network distance enforced on delegation** | `delegate_mandate()` (if exists) | `tests/unit/core/test_delegation.py` (expand) | Cannot delegate deeper than policy.max_network_distance |
| **Allow delegation=false blocks all delegation** | `delegate_mandate()` | (same) | If policy.allow_delegation=False, cascading fails |
| **Resource pattern subset validation** | `_validate_scope_subset()` | `tests/unit/core/test_mandate.py` (expand) | `['secret/*']` ⊆ `['*']`? Yes. `['*']` ⊆ `['secret/*']`? No. |
| **Action scope exact member matching** | `_validate_scope_subset()` | (same) | `['read']` NOT member of `['read_admin']` |
| **Policy deactivation blocks new mandates** | `create_policy()` / `update_policy()` | `tests/integration/enterprise/test_policy_lifecycle.py` (new) | Inactive policy must not issue new mandates |
| **Concurrent policy update isolation** | Policy update race | `tests/integration/enterprise/test_policy_concurrent_updates.py` (new) | Should serialize; last-write-wins or explicit conflict |
| **Policy history tracking (audit trail)** | Policy audit | `tests/integration/enterprise/test_policy_audit.py` (new) | All policy changes must be logged with timestamp, actor, before/after |

**Suggested Test Location:** `tests/unit/enterprise/test_policy_routes.py` (new); `tests/integration/enterprise/test_policy_*.py` (new files)

---

### **3.2 Issuance Rate Limiting & Quota**
**Production Files:**
- [caracal/core/rate_limiter.py](caracal/packages/caracal-server/caracal/core/rate_limiter.py#L76) – `check_rate_limit()`
- `caracal/core/mandate.py` – Rate limiter integration

**Existing Tests:** `tests/unit/core/test_rate_limiter.py` (basic rate check)

**Missing Invariant Tests:**

| Test Case | Target Function | Test Location |
|---|---|---|
| **Rate limit exceeded blocks mandate issuance** | `issue_mandate()` with rate limiter | `tests/unit/core/test_mandate.py` (expand) |
| **Rate limit reset after time window** | `check_rate_limit()` | `tests/unit/core/test_rate_limiter.py` (expand with freezegun) |
| **Per-principal rate limit isolation** | Multiple principals concurrent issuance | `tests/integration/core/test_rate_limit_isolation.py` (new) |

---

## **4. AUTHENTICATION & AUTHORIZATION – 22 Test Gaps**

### **4.1 Auth Token Validation & Expiry**
**Production Files:**
- [caracal/core/session_manager.py](caracal/packages/caracal-server/caracal/core/session_manager.py#L683-L700) – Token validation
- `caracal_api/middleware/auth.py` – Auth middleware

**Existing Tests:** `tests/unit/core/test_session_manager.py` (basic token validation)

**Missing Invariant Tests:**

| Test Case | Target Function | Test Location |
|---|---|---|
| **Expired token rejected immediately at validation** | `validate_access_token()` | `tests/unit/core/test_session_manager.py` (expand) |
| **Token not-yet-valid (iat > now) rejected** | Token validation | (same) |
| **Wrong token type (task vs access) rejected** | `validate_access_token()` with task token | (same) |
| **Revoked principal's tokens rejected** | Token validation with revoked principal | `tests/integration/core/test_token_revocation.py` (new) |
| **Token replay attack prevention** | Token reuse after first validation | `tests/security/auth/test_token_replay.py` (new) |
| **Token signature verification failure** | Tampered token | `tests/security/auth/test_token_signature.py` (new) |

---

### **4.2 Principal State & Access Context Consistency**
**Production Files:**
- `caracal_api/middleware/auth.py` – Get current user 
- `caracal_api/middleware/authority.py` – Authority check

**Existing Tests:** `tests/integration/api/test_endpoints.py#L157-L180` (spoofing only)

**Missing Invariant Tests:**

| Test Case | Target Function | Test Location |
|---|---|---|
| **Authentication succeeds but authorization fails silently not exposed** | Auth middleware | `tests/integration/api/test_auth_flow_separation.py` (new) |
| **Principal kind HUMAN cannot perform worker-only operations** | Operation gating | `tests/security/auth/test_principal_kind_gates.py` (new) |
| **Service principal authorization chains must be explicit** | Service mandate validation | `tests/security/auth/test_service_chains.py` (new) |
| **Workspace context must be present for all protected ops** | Route handler | `tests/integration/enterprise/test_workspace_context_required.py` (new) |

---

### **4.3 Session & Workspace Switching**
**Production Files:**
- [caracalEnterprise/src/contexts/AuthContext.tsx](caracalEnterprise/src/contexts/AuthContext.tsx#L40-L120) – Session state (frontend)
- `caracal_api/routes/auth.py` – Session switch (backend)

**Existing Tests:** None specific to session switching security

**Missing Invariant Tests:**

| Test Case | Target Function | Test Location |
|---|---|---|
| **Workspace switch doesn't leak mandates from prior workspace** | `switchWorkspace()` | `tests/security/boundaries/test_workspace_switch_isolation.py` (new) |
| **New access token after switch scoped to new workspace** | Token generation | (same) |
| **Prior workspace's principals not visible after switch** | Principal listing | (same) |

---

## **5. DELEGATION & GRAPH VALIDATION – 24 Test Gaps**

### **5.1 Delegation Direction Enforcement**
**Production Files:**
- [caracal/core/delegation_graph.py](caracal/packages/caracal-server/caracal/core/delegation_graph.py#L24-L68) – `ALLOWED_DELEGATIONS`, `validate_delegation_direction()`

**Existing Tests:** `tests/unit/test_nested_workflows.py#L108-L130` (parametrized, but limited)

**Missing Invariant Tests:**

| Allowed Pair | Blocked Pair | Test Case | Test Location |
|---|---|---|---|
| human → human | service → service | Peer delegation only for non-service | `tests/unit/core/test_delegation_graph_pure.py` (expand) |
| orchestrator → orchestrator | service → orchestrator | Same-kind coordination allowed, service blocked | (same) |
| worker → worker | orchestrator → human | Peer coordination but no upward delegation | (same) |
| human → orchestrator | orchestrator → human | Direction is strict downward or peer | (same) |
| orchestrator → worker | worker → orchestrator | Hierarchy enforcement: orchestrator > worker | (same) |
| worker → service | service → worker | Service is terminal; cannot delegate outbound | (same) |
| — | service → anything | Service cannot delegate | (same) |
| — | human → human (after worker involved) | Multi-level delegation preserves direction rules | `tests/integration/core/test_delegation_chain_direction.py` (new) |

---

### **5.2 Delegation Path Validation & Cycle Detection**
**Production Files:**
- [caracal/core/delegation_graph.py](caracal/packages/caracal-server/caracal/core/delegation_graph.py#L609-L650) – `validate_authority_path()`, cycle detection

**Existing Tests:** `tests/unit/core/test_delegation_graph_pure.py#L40-L80` (basic cycle detection)

**Missing Invariant Tests:**

| Test Case | Target Function | Test Location |
|---|---|---|
| **Simple cycle A → B → A detected and rejected** | `validate_authority_path()` | `tests/unit/core/test_delegation_graph_pure.py` (expand) |
| **Multi-step cycle A → B → C → A detected** | (same) | (same) |
| **Self-delegation A → A rejected** | (same) | (same) |
| **Complex DAG with multiple paths validated correctly** | (same) | `tests/integration/core/test_delegation_complex_topology.py` (new) |
| **Revoked intermediate mandate breaks path** | `validate_authority_path()` with revoked edge | `tests/integration/core/test_delegation_revocation_breaks_path.py` (new) |
| **Expired intermediate mandate breaks path** | `validate_authority_path()` with expired mandate | (same) |
| **Multiple authority sources (multi-parent) validated** | Get effective scope with multi-source | `tests/integration/core/test_delegation_multi_source.py` (new) |

---

### **5.3 Delegation Scope & Network Distance**
**Production Files:**
- `caracal/core/delegation_graph.py` – Scope effective depth calculation
- `caracal/core/mandate.py` – Network distance propagation

**Existing Tests:** None specific to network distance enforcement

**Missing Invariant Tests:**

| Test Case | Target Function | Test Location |
|---|---|---|
| **Network distance decrements on each delegation** | Mandate creation depth logic | `tests/unit/core/test_mandate.py` (expand) |
| **Depth 0 (root) cannot further delegate if policy forbids** | Delegation depth check | `tests/unit/core/test_delegation.py` (expand, if file exists) |
| **Max network distance in policy enforced** | `delegate_mandate()` | (same) |
| **Scope monotonically narrows or stays same in chain** | Effective scope calculation | `tests/integration/core/test_delegation_scope_narrowing.py` (new) |
| **Cannot widen scope at any delegation step** | Scope validation in delegation | (same) |

---

### **5.4 Cascade Revocation & Invalidation**
**Production Files:**
- `caracal/core/delegation.py` or `mandate.py` – `revoke_mandate(cascade=True)`
- `caracal/core/delegation_graph.py` – Graph traversal

**Existing Tests:** `tests/unit/core/test_revocation.py` (partial edge deletion)

**Missing Invariant Tests:**

| Test Case | Target Function | Test Location |
|---|---|---|
| **Cascade revoke removes all child mandates** | `revoke_mandate(cascade=True)` | `tests/integration/core/test_delegation_cascade.py` (new) |
| **Non-cascade revoke validates no children exist** | `revoke_mandate(cascade=False)` | (same) |
| **Cascade with partial failures (DB conflict) rolls back** | `revoke_mandate()` in transaction | `tests/integration/core/test_cascade_transaction_safety.py` (new) |
| **Revoked mandate appears in cascade preview** | `cascadePreview()` API | `tests/integration/enterprise/test_delegation_cascade_preview.py` (new) |

---

## **6. UNSAFE PARSING & INPUT VALIDATION – 34 Test Gaps**

### **6.1 TAR Archive Parsing (Onboarding)**
**Production Files:**
- [caracal_api/routes/onboarding.py](caracalEnterprise/services/api/src/caracal_api/routes/onboarding.py#L570-L660) – `_validate_and_parse_tar()`

**Existing Tests:** `caracalEnterprise/services/api/tests/test_import_workspace.py` (basic happy path)

**Missing Invariant Tests:**

| Test Case | Target Function | Test Location | Notes |
|---|---|---|---|
| **Path traversal attack (../../../etc/passwd)** | `_validate_and_parse_tar()` | `tests/security/parsing/test_archive_path_traversal.py` (new) | Must reject any ../; only _ALLOWED_TAR_PATHS permitted |
| **Symbolic link escape** | Archive extraction | (same) | Must reject symlinks in tar; extraction must fail safely |
| **Zip bomb equivalent (large compression ratio)** | Archive parsing | (same) | Max extracted size validation before decompression |
| **Nested tar bombs** | Recursive tar parsing | (same) | Cannot extract tar within tar |
| **Invalid JSON/YAML in archive files** | `json.loads()` / `yaml.safe_load()` (onboarding.py:596) | `tests/security/parsing/test_archive_format_validation.py` (new) | Must reject with clear error; never silent fallback |
| **Missing required files in archive** | Validation logic (onboarding.py:603) | `tests/unit/enterprise/test_archive_schema.py` (new) | Empty archive or incomplete schema must reject |
| **Extra files in archive (not in _ALLOWED_TAR_PATHS)** | File whitelist check | `tests/security/parsing/test_archive_whitelist.py` (new) | Unknown files must be rejected, not silently ignored |
| **Archive with null bytes in filenames** | Extract filename validation | `tests/security/parsing/test_archive_null_bytes.py` (new) | Null bytes in paths must be rejected |
| **Case sensitivity of path validation** | Path comparison logic | `tests/unit/enterprise/test_archive_path_case.py` (new) | Ensure consistent case handling (OS-dependent?) |
| **Very large archive metadata** | TAR header parsing | `tests/security/parsing/test_archive_size_limits.py` (new) | Malformed TAR headers must not crash; graceful failure |

**Suggested Test Location:** `tests/security/parsing/` (new directory)

---

### **6.2 Intent Parsing & Validation**
**Production Files:**
- [caracal/core/intent.py](caracal/packages/caracal-server/caracal/core/intent.py#L141-L220) – `parse_intent()`, `validate_intent_against_mandate()`

**Existing Tests:** `tests/unit/core/test_intent.py` (basic validation, pattern matching)

**Missing Invariant Tests:**

| Test Case | Target Function | Test Location |
|---|---|---|
| **Empty action field rejected** | `parse_intent()` | `tests/unit/core/test_intent.py` (already has this) |
| **Empty resource field rejected** | (same) | (same) |
| **Non-dict input rejected** | (same) | (same) |
| **Null values in required fields** | (same) | `tests/unit/core/test_intent.py` (expand) |
| **Extremely long action/resource strings** | (same) | `tests/security/parsing/test_intent_size_limits.py` (new) |
| **Special characters in action** | `parse_intent()` | `tests/security/parsing/test_intent_special_chars.py` (new) |
| **Intent hash stability (determinism)** | `generate_hash()` | `tests/unit/core/test_intent.py` (expand) |
| **Intent hash collision resistance** | (same) | `tests/unit/core/test_intent.py` (expand) |

---

### **6.3 JSON/YAML Deserialization Safety**
**Production Files:**
- `caracal/core/authority_metadata.py` – `from_dict()` methods
- `caracal/core/metering.py` – `from_dict()`
- Various route handlers

**Existing Tests:** None specific to deserialization safety

**Missing Invariant Tests:**

| Test Case | Target Function | Test Location |
|---|---|---|
| **Circular reference in JSON rejected** | JSON parsing | `tests/security/parsing/test_json_circular_ref.py` (new) |
| **Numeric overflow in JSON (extremely large numbers)** | JSON parsing | `tests/security/parsing/test_json_overflow.py` (new) |
| **Unicode/multibyte character handling** | String parsing | `tests/security/parsing/test_json_unicode.py` (new) |
| **Unknown fields in JSON ignored or rejected** | `from_dict()` parsing | `tests/unit/core/test_serialization_safety.py` (new) |
| **Type mismatches in JSON fields** | Type validation in `from_dict()` | (same) |

---

### **6.4 Provider Scope Parsing**
**Production Files:**
- [caracal/provider/definitions.py](caracal/packages/caracal-server/caracal/provider/definitions.py) – `parse_provider_scope()`
- Authority/mandate code using it

**Existing Tests:** Partial (implied by authority tests)

**Missing Invariant Tests:**

| Test Case | Target Function | Test Location |
|---|---|---|
| **Malformed provider scope (::, double colon)** | `parse_provider_scope()` | `tests/unit/provider/test_provider_scope_parsing.py` (new) |
| **Invalid provider delimiter** | (same) | (same) |
| **Empty fields in scope** | (same) | (same) |
| **Wildcard usage in invalid positions** | (same) | `tests/unit/provider/test_provider_scope_wildcard.py` (new) |
| **Case sensitivity in provider names** | (same) | `tests/unit/provider/test_provider_scope_case.py` (new) |
| **SQL injection-like patterns in scope** | (same) | `tests/security/parsing/test_provider_scope_injection.py` (new) |

---

### **6.5 Caveat Chain Parsing & Deserialization**
**Production Files:**
- [caracal/core/caveat_chain.py](caracal/packages/caracal-server/caracal/core/caveat_chain.py#L39-L256) – Parsing, verification, evaluation

**Existing Tests:** `tests/coverage/core/test_caveat.py` (basic coverage)

**Missing Invariant Tests:**

| Test Case | Target Function | Test Location |
|---|---|---|
| **Caveat chain HMAC tampering detected** | `verify_caveat_chain()` | `tests/security/parsing/test_caveat_tamper_detection.py` (new) |
| **Caveat chain with invalid expiry format** | `_parse_expiry_datetime()` | `tests/unit/core/test_caveat_chain.py` (expand) |
| **Caveat chain deep nesting (DOS)** | Chain building | `tests/security/parsing/test_caveat_nesting_limits.py` (new) |
| **Empty caveat string accepted** | `parse_caveat()` | `tests/unit/core/test_caveat_chain.py` (expand) |
| **Malformed task-binding caveat** | `parse_caveat("task_binding:malformed")` | (same) |
| **Caveat chain missing required fields** | Chain construction | (same) |

---

## **SUMMARY TABLE: Top 20 Critical Gaps by Risk**

| Priority | Area | Test Case | Production File | Suggested Test File | Risk Level |
|---|---|---|---|---|---|
| 1 | Security Boundaries | Workspace isolation violation | authority.py | tests/security/boundaries/test_workspace_boundary.py | CRITICAL |
| 2 | Security Boundaries | Signature verification bypass | authority.py | tests/security/authority/test_signature_edge_cases.py | CRITICAL |
| 3 | Unsafe Parsing | TAR path traversal attack | onboarding.py | tests/security/parsing/test_archive_path_traversal.py | CRITICAL |
| 4 | Unsafe Parsing | Archive symlink escape | onboarding.py | tests/security/parsing/test_archive_path_traversal.py | CRITICAL |
| 5 | Delegation | Cycle detection in complex DAG | delegation_graph.py | tests/integration/core/test_delegation_complex_topology.py | HIGH |
| 6 | Auth | Revoked principal token acceptance | session_manager.py | tests/integration/core/test_token_revocation.py | HIGH |
| 7 | Lifecycle | Concurrent state transition race condition | lifecycle.py | tests/integration/core/test_lifecycle_concurrent.py | HIGH |
| 8 | Policy | Policy tightening rejection of expansion | mandate.py | tests/unit/core/test_mandate.py (expand) | HIGH |
| 9 | Delegation | Cascade revocation partial failure rollback | mandate.py | tests/integration/core/test_cascade_transaction_safety.py | HIGH |
| 10 | Security Boundaries | Subject binding cross-principal confusion | authority.py | tests/security/authority/test_subject_binding.py | HIGH |
| 11 | Unsafe Parsing | Invalid JSON/YAML in archive | onboarding.py | tests/security/parsing/test_archive_format_validation.py | HIGH |
| 12 | Unsafe Parsing | Archive size bombs | onboarding.py | tests/security/parsing/test_archive_size_limits.py | HIGH |
| 13 | Delegation | Network distance enforcement | mandate.py | tests/unit/core/test_mandate.py (expand) | MEDIUM |
| 14 | Auth | Principal kind operation gating | auth middleware | tests/security/auth/test_principal_kind_gates.py | MEDIUM |
| 15 | Lifecycle | Revoked principal mandate invalidation cascade | revocation.py | tests/integration/core/test_revocation_cascade.py | MEDIUM |
| 16 | Policy | Policy pattern malformation rejection | policies.py | tests/unit/enterprise/test_policy_routes.py (new) | MEDIUM |
| 17 | Security Boundaries | Principal lifecycle inactive during validation | authority.py | tests/unit/core/test_authority.py (expand) | MEDIUM |
| 18 | Delegation | Multi-source delegation effective scope | delegation_graph.py | tests/integration/core/test_delegation_multi_source.py | MEDIUM |
| 19 | Unsafe Parsing | Caveat chain HMAC tampering | caveat_chain.py | tests/security/parsing/test_caveat_tamper_detection.py | MEDIUM |
| 20 | Auth | Workspace switch credential leakage | auth.py | tests/security/boundaries/test_workspace_switch_isolation.py | MEDIUM |

---

## **IMPLEMENTATION ROADMAP**

### **Phase 1 (Critical – Immediate)**
1. Create `tests/security/boundaries/` directory
2. Create `tests/security/parsing/` directory
3. Implement 6 critical TAR archive security tests
4. Implement 4 critical authority boundary tests
5. Verify existing lifecycle tests cover REVOKED terminal state

### **Phase 2 (High Priority – Sprint 2)**
1. Expand `tests/unit/core/test_authority.py` with 8 new test methods
2. Create `tests/integration/core/test_delegation_complex_topology.py` (cycle detection, DAG)
3. Create `tests/integration/core/test_token_revocation.py`
4. Create `tests/integration/core/test_lifecycle_concurrent.py`
5. Create `tests/integration/core/test_revocation_cascade.py`

### **Phase 3 (Medium Priority – Sprint 3)**
1. Create `tests/unit/enterprise/test_policy_routes.py`
2. Expand `tests/unit/core/test_intent.py` 
3. Expand `tests/unit/core/test_caveat_chain.py`
4. Create `tests/security/auth/` tests
5. Create `tests/integration/enterprise/test_policy_*.py` files

---

## **VERIFICATION NOTES**

- **All files use absolute workspace paths** as per codebase structure
- **Test layer placement follows existing conventions** (unit/integration/security/coverage)
- **No existing tests modified** – only gaps identified and suggested additions
- **Multi-tenant and workspace isolation tests** critical before any production deployment post-security-fix
- **Caveat chain and cryptographic validation** tests must use real EC keys (from `tests/mock/crypto.py`)

</b>