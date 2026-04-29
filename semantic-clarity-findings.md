# Caracal Semantic Clarity & Edge Case Analysis

## PART A: Semantic Clarity Issues

### Terminology Overloading
| Term | Used For | Contexts | Issue |
|------|----------|----------|-------|
| `mandate` | ExecutionMandate (auth token) | Core, Enterprise, Ledger, Delegation | ✓ Consistent |
| `principal` | Identity entity (human/orchestrator/worker/service) | Everywhere | ✓ Consistent |
| `subject` | Target of authority (in mandate) | Authority, Delegation | ✓ Consistent but sometimes confused with issuer |
| `issuer` | Source of authority (in mandate) | Authority, Delegation | ✓ Consistent |
| `scope` | Resource/action scope arrays | Mandate, Authority Policy, Delegation | Ambiguous: resource_scope vs action_scope treated same way |
| `caveat` | Restriction chain | caveat_chain.py only | ✓ Clear but terminology "caveat_chain" uncommon |
| `network_distance` | Delegation hops limit | Authority Policy | ✓ Clear (max_network_distance) |
| `intent_hash` | SHA-256(action+resource+params) | Mandate | ✓ Clear |

### Inconsistent Enum/Status Names
| Model | Status Field | Values | Issue |
|-------|-------------|--------|-------|
| Principal | `lifecycle_status` | PENDING_ATTESTATION, ACTIVE, SUSPENDED, DEACTIVATED, EXPIRED, REVOKED | ✓ Consistent |
| ExecutionMandate | `revoked` (boolean) | true/false | Not lifecycle status (OK for mandates) |
| ExecutionMandate | No explicit "expired" status | Checked via valid_from/valid_until | Valid but not explicit |
| Frontend | `lifecycle_status` | "active" \| "suspended" \| "deactivated" \| "revoked" (missing PENDING_ATTESTATION, EXPIRED) | **Mismatch: Missing PENDING_ATTESTATION and EXPIRED** |

### Unclear Concept Boundaries
1. **AuthorityPolicy vs ResourceAllowlist**: Both control resource access but unclear when each applies
   - AuthorityPolicy: per-principal issuance constraints (allowed_resource_patterns)
   - ResourceAllowlist: separate table for per-principal resource restrictions
   - **Overlap risk**: Two parallel restriction mechanisms

2. **Mandate Scope vs AuthorityPolicy**: 
   - Mandate has resource_scope + action_scope
   - AuthorityPolicy has allowed_resource_patterns + allowed_actions
   - Relationship: Policy constrains what can be IN mandates, but mandate scopes must still match

3. **Delegation vs Caveat**: Both narrow scope
   - Delegation narrows via source mandate's scopes
   - Caveat narrows via HMAC chain restrictions
   - **Overlap**: Both cumulative, but caveats are append-only HMAC-bound

## PART B: Edge Cases & Assumptions

### Critical Findings

#### 1. **TIMEZONE MISMATCH** - HIGH SEVERITY
Files affected:
- `caracalEnterprise/services/gateway/compliance_events.py:34,36` - Uses `datetime.utcnow()` (deprecated, naive)
- `caracalEnterprise/services/gateway/gateway_client.py:106,130,151` - Uses `datetime.now()` (naive)
- `caracalEnterprise/services/api/integrations/caracal_core.py:90+` - Uses `datetime.utcnow()`
- `Caracal/packages/caracal-server/caracal/db/models.py:1024` - Uses `datetime.utcnow()`

Impact: Clock skew in expiry checks; comparison of naive vs aware datetimes raises TypeError

#### 2. **SUSPENDED PRINCIPAL CAN STILL ISSUE MANDATES** - MEDIUM SEVERITY
Location: `caracal/core/authority.py:507` only checks SUBJECT principal is ACTIVE
- ISSUER principal's lifecycle_status is NOT checked
- A SUSPENDED/REVOKED principal can still issue new mandates
- Repro: Create principal → SUSPEND it → issue_mandate() succeeds

#### 3. **EMPTY SCOPES ALWAYS COVERED** - MEDIUM SEVERITY
Location: `caracal/core/delegation_graph.py:_scope_is_covered_by_union()`
```python
# Line 320: for requested_entry in requested_scope or []:
#   if not any(...): return False
# return True  # <- returns True if requested_scope is [] or None
```
- Edge case: Both source and target have empty scopes `[]`
- Result: Delegation allowed (caveat validation passes vacuously)
- Impact: Overly permissive; should require explicit scopes or catch this

#### 4. **INTENTHASH MISMATCH NOT CAUGHT** - MEDIUM SEVERITY
- Mandate can have `intent_hash` set
- Request validation never checks if request's intent matches mandate's intent_hash
- Current code: no validation of intent against mandate.intent_hash in authority.py
- Risk: Intent constraint is silently ignored

#### 5. **NO CYCLE/LOOP DETECTION IN DELEGATION GRAPH** - LOW SEVERITY
- `validate_authority_path()` uses `visited` set but no explicit cycle check
- Peer delegations (A↔B↔A) are allowed
- Risk: Infinite recursion if graph has cycles (unlikely given DB constraints but unfenced)

#### 6. **CONCURRENT REVOCATION + VALIDATION RACE** - MEDIUM SEVERITY
- Mandate validation checks `revoked` flag
- Between check and use, mandate could be revoked
- No transaction-level optimistic locking
- Race window: ~ns to seconds depending on load

#### 7. **MANDATE CASCADE ASYNC FAILURE NOT HANDLED** - LOW SEVERITY
Location: `caracal/core/revocation.py:144+`
- If cascade of >250 mandates starts async and process dies mid-cascade
- Revocation events published but cascade incomplete
- Ledger shows revocation but dependent mandates still "active" in DB
- Recovery mechanism: unclear

#### 8. **PRINCIPAL KIND IMMUTABILITY NOT ENFORCED** - LOW SEVERITY
- `principal_kind` set at creation; no SQL constraint preventing UPDATE
- Update via `principal.update()` or direct DB access could change kind
- Impact: Delegation rules may be violated post-creation
- Mitigation: Application-level guard but no DB-level constraint

#### 9. **EXPIRED PRINCIPALS RE-ACTIVATABLE** - LOW SEVERITY
Location: `caracal/core/lifecycle.py:51`
- EXPIRED → ACTIVE transition allowed (for humans/services)
- Orchestrators/workers cannot reactivate from EXPIRED
- Semantics: EXPIRED is "timed-out", not "deactivated by admin"
- Risk: Unclear when re-activation is appropriate; no retention policy

#### 10. **VAULT UNREACHABLE BEHAVIOR** - MEDIUM SEVERITY
Config: `CCL_PRINCIPAL_KEY_BACKEND=vault` (hardcut, no fallback)
- If Vault is unreachable during mandate validation (signature check)
- Fail-closed: validation throws, decision is DENY (good)
- But: No explicit logging/monitoring that Vault is unreachable
- Risk: Silent degradation hard to debug

#### 11. **EMPTY CAVEAT CHAIN vs MISSING CHAIN** - LOW SEVERITY
Location: `caracal/core/caveat_chain.py`
- `caveat_chain = []` means no restrictions (all allowed)
- `caveat_chain = None` same as `[]`
- Semantics: Should empty chain be different from "no caveat"?
- Current: Both treated identically (vacuously true)

#### 12. **PRINCIPAL KIND MISMATCH IN FRONTEND** - LOW SEVERITY
Frontend types missing PENDING_ATTESTATION and EXPIRED statuses:
`src/types/caracal.ts:27`: `"active" | "suspended" | "deactivated" | "revoked"`
Should be: `"pending_attestation" | "active" | "suspended" | "expired" | "deactivated" | "revoked"`

