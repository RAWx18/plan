# Trust/Authority/Delegation Audit - Session Notes

## Lifecycle Phases Identified

### 1. REGISTRATION → DELEGATION → ENFORCEMENT → VERIFICATION → REVOCATION

**Participants:**
- Principals (4 kinds: human, orchestrator, worker, service)
- ExecutionMandates (signed authority tokens)
- AuthorityPolicies (delegation rules)
- AuthorityLedgerEvents (audit trail)
- DelegationEdges (delegation graph)

**Key Files:**
- caracal/core/mandate.py - Issuance/revocation
- caracal/core/authority.py - Validation
- caracal/core/revocation.py - Revocation orchestration  
- caracal_api/middleware/authority.py - API enforcement
- caracal_api/middleware/rbac.py - RBAC layer

## Critical Gaps Found

### HIGH SEVERITY

1. **Client-Side Permission Enforcement Unvalidated**
   - File: caracalEnterprise/src/lib/permissions.ts
   - Issue: hasPagePermission() is client-side only; no server validation
   - Impact: UI controls bypassed=auth bypassed
   - Rewrite/Patch: Patch - add server-side middleware

2. **Admin Bypass in Principal Revocation**
   - File: caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py:L290-320
   - Issue: is_admin allows bypassing revoker validation
   - Impact: Non-admin revokers can escalate via admin flag
   - Rewrite/Patch: Rewrite - remove admin bypass path

3. **Missing Workspace Isolation in Registration**
   - File: caracal/cli/principal.py - principal registration
   - Issue: No workspace scoping during CLI registration
   - Impact: Principals can escape workspace boundaries
   - Rewrite/Patch: Patch - add owner/workspace validation

### MEDIUM SEVERITY

4. **RBAC and Mandate Authority Both Present**
   - Two parallel auth systems without clear priority
   - File: caracalEnterprise/services/api/src/caracal_api/middleware/
   - Issue: AuthorityCheck and RBAC both used; no explicit ordering
   - Impact: Conflicting enforcement points create gaps
   - Rewrite/Patch: Rewrite - consolidate into single system

5. **Configuration Loading Without Validation**
   - File: caracal/config/settings.py
   - Issue: Defaults used without validation of consistency
   - Impact: Malformed defaults become implicit trust
   - Rewrite/Patch: Patch - add startup validation

6. **Missing Explicit Cascade Enforcement**
   - File: caracalEnterprise/services/api/src/caracal_api/routes/delegation.py
   - Issue: cascade parameter optional; some revocation paths don't enforce it
   - Impact: Delegation chains can survive revocation
   - Rewrite/Patch: Patch - make cascade required

### IMPLICIT TRUST ISSUES

7. **Service-to-Service Auth Not Explicit**
   - Files: caracalEnterprise/services/api/src/caracal_api/ (multiple)
   - Issue: Gateway, API, Vault communicate without mutual auth proof
   - Impact: Service compromise = system compromise
   - Rewrite/Patch: Rewrite - add mTLS/JWT service-to-service

8. **Attestation Status Unchecked in Enforcement**
   - File: caracal/core/authority.py - validate_mandate_state()
   - Issue: Principal.attestation_status field never checked
   - Impact: Unattested principals can exercise authority
   - Rewrite/Patch: Patch - enforce attestation in validation

## Delegation Depth Enforcement

- Network_distance tracked in ExecutionMandate
- Validated in mandate.delegate_mandate() - GOOD
- But delegated_depth calculation has edge case: source_depth <= 0 prevents ANY delegation
- Should be source_depth > 0, not <= 0

## Revocation Enforcement

**Good:**
- PrincipalRevocationOrchestrator uses leaves-first traversal
- Cascade revocation implemented in delegation_graph.py
- Authority ledger records all revocation events

**Gaps:**
- Cascade parameter optional in revoke endpoints
- is_admin bypasses revoker validation
- Session denylist not universally enforced

## Verification Checklist

- Signature verification: Present in authority.py _validate_issuer_authority()
- Expiry checking: Present but relies on datetime.utcnow()
- Scope matching: Pattern matching implemented with fnmatch
- Revocation status: Checked in _validate_mandate_state()
- Lifecycle status: Checks principal.lifecycle_status but not attestation_status
