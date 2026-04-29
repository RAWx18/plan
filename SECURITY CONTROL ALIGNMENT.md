Based on my comprehensive analysis of both repositories, here is the **Enterprise vs OSS Security Drift Report**:

---

## SECURITY CONTROL ALIGNMENT TABLE

| Control | OSS Implementation | Enterprise Implementation | Status | Risk Level | Fix Action |
|---------|-------------------|---------------------------|--------|------------|-----------|
| **1. JWT Validation** | [session_manager.py](../Caracal/packages/caracal-server/caracal/core/session_manager.py) `SessionManager.validate_access_token()` with asymmetric signing (RS256/ES256), deny-list check, caveat chain support | [auth.py](../caracalEnterprise/services/api/src/caracal_api/utils/auth.py#L87) calls OSS `get_session_manager()` - **ALIGNED** | ✅ ALIGNED | LOW | N/A |
| **2. Principal Lifecycle Enforcement** | [lifecycle.py](../Caracal/packages/caracal-server/caracal/core/lifecycle.py) `PrincipalLifecycleStateMachine` with typed state transitions, non-reactivating worker/orchestrator principals | Enterprise does **NOT** import or use `PrincipalLifecycleStateMachine`. Routes accept `lifecycle_status` via API but skip state machine validation | 🔴 **CRITICAL DRIFT** | **HIGH** | **ALIGN_ENTERPRISE** - Import and enforce OSS state machine before any lifecycle mutations |
| **3. Authority Decision Validation** | [authority.py](../Caracal/packages/caracal-server/caracal/core/authority.py#L115) `AuthorityEvaluator.validate_mandate()` with boundary stages, ledger recording | [caracal_core.py](../caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py#L77) instantiates OSS `AuthorityEvaluator` - **ALIGNED** | ✅ ALIGNED | LOW | N/A |
| **4. Session Denial List** | [session_manager.py](../Caracal/packages/caracal-server/caracal/core/session_manager.py#L96) `RedisSessionDenylistBackend` with token JTI tracking + principal revocation timestamps | [auth.py](../caracalEnterprise/services/api/src/caracal_api/middleware/auth.py#L99) calls `is_token_blacklisted()` and `is_principal_session_revoked()` - uses same Redis backend | ✅ ALIGNED | LOW | N/A |
| **5. Vault Backend Enforcement** | [vault.py](../Caracal/packages/caracal-server/caracal/core/vault.py#L826) raises `VaultError` when vault unreachable in hardcut mode. OSS forbids local/file backends. | Enterprise [auth.py](../caracalEnterprise/services/api/src/caracal_api/utils/auth.py#L28) uses `VaultReferenceJwtSigner` for session keys. [main.py](../caracalEnterprise/services/api/src/caracal_api/main.py#L283) runs `assert_enterprise_hardcut()` at startup | ✅ ALIGNED | LOW | N/A |
| **6. Workspace Isolation** | OSS uses `Principal.owner` field for multi-tenancy. No query-level workspace enforcement (single-tenant assumption) | Enterprise [caracal_core.py](../caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py#L145) adds `Principal.owner == self.workspace_id` filter on **all** queries. Workspace mismatch = hard denial before authority eval | ✅ **STRONGER IN ENTERPRISE** | LOW | **INTENTIONAL_DIFFERENCE** - Enterprise adds multi-tenant safety layer |
| **7. Rate Limiting** | [rate_limiter.py](../Caracal/packages/caracal-server/caracal/core/rate_limiter.py) `MandateIssuanceRateLimiter` with sliding window (100/hour, 10/min per principal). **Fail-closed** on Redis errors. | [rate_limit.py](../caracalEnterprise/services/api/src/caracal_api/middleware/rate_limit.py) FastAPI middleware with **fixed window** (20/min auth, 60/min write, 300/min read, 1000/min validate). **Fail-open** on Redis errors | ⚠️ **INCONSISTENT** | **MEDIUM** | **ALIGN_ENTERPRISE** - Use fail-closed semantics. Consider harmonizing limits or documenting divergence |
| **8. Audit Logging** | [audit.py](../Caracal/packages/caracal-server/caracal/core/audit.py) `AuditLogManager` with cryptographic hash chains, tamper detection. Authority decisions logged via `AuthorityEvaluator._record_ledger_event()` | Enterprise relies on OSS audit layer. No additional enterprise-layer audit enrichment. Authority decisions flow through OSS `AuthorityEvaluator` | ✅ ALIGNED | LOW | N/A |
| **9. Error Responses** | [exceptions.py](../Caracal/packages/caracal-server/caracal/exceptions.py) raises typed exceptions (`PrincipalNotFoundError`, `PolicyError`, etc.). CLI/SDK catch and format. | [errors.py](../caracalEnterprise/services/api/src/caracal_api/utils/errors.py) uses FastAPI `HTTPException` with structured JSON: `{"detail": {"code": "...", "message": "...", "guidance": "..."}}`. Different error codes. | ⚠️ **INCONSISTENT** | LOW | **INTENTIONAL_DIFFERENCE** - REST API needs HTTP-native errors. Document mapping. |
| **10. Hardcut Preflight** | [hardcut_preflight.py](../Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py) enforces: vault-only secrets, asymmetric session signing, no SQLite, no local state files | Enterprise [main.py](../caracalEnterprise/services/api/src/caracal_api/main.py#L203) calls `assert_migration_hardcut()` and `assert_enterprise_hardcut()` at startup. Custom check `_assert_registration_state_hardcut_contract()` | ✅ ALIGNED | LOW | N/A |
| **11. Principal Attestation** | OSS [authority.py](../Caracal/packages/caracal-server/caracal/core/authority.py#L66) checks `PrincipalAttestationStatus` and denies if not attested (`AUTH_PRINCIPAL_NOT_ATTESTED`) | Enterprise does **NOT** check attestation status in routes. Relies on OSS `AuthorityEvaluator` to enforce during mandate validation, but no API-layer guard | ⚠️ **DRIFT** | **MEDIUM** | **ALIGN_ENTERPRISE** - Add attestation checks in principal mutation routes |
| **12. Privileged Re-Auth** | Not present in OSS (CLI/TUI are single-session, no step-up auth) | Enterprise [auth.py](../caracalEnterprise/services/api/src/caracal_api/middleware/auth.py#L221) has `require_privileged_reauth()` with configurable threshold (900s default) and optional MFA requirement | ✅ **STRONGER IN ENTERPRISE** | LOW | **INTENTIONAL_DIFFERENCE** - Enterprise adds step-up auth for web |

---

## CRITICAL DRIFT FINDINGS

### 🔴 **#1: Principal Lifecycle State Machine Bypass (HIGH RISK)**

**Location:** [principals.py](../caracalEnterprise/services/api/src/caracal_api/routes/principals.py#L360)

**OSS Behavior:**
- [lifecycle.py](../Caracal/packages/caracal-server/caracal/core/lifecycle.py#L91) enforces typed state transitions
- Orchestrator/Worker principals are **non-reactivating** (cannot go from `deactivated` → `active`)
- Human principals can transition `suspended` → `active` but orchestrators cannot
- Attestation requirement: Orchestrator/Worker must be `attested` before `pending_attestation` → `active`

**Enterprise Behavior:**
```python
# routes/principals.py line 360
if principal_data.lifecycle_status:
    status=principal_data.lifecycle_status  # ← DIRECT UPDATE, NO STATE MACHINE
```

**Risk:**
- Enterprise API allows **invalid lifecycle transitions** (e.g., revoked → active orchestrator)
- Bypasses hard-cut non-reactivation policy for worker/orchestrator principals
- Violates attestation-gated activation

**Fix:**
```python
from caracal.core.lifecycle import PrincipalLifecycleStateMachine, LifecycleTransitionError

lifecycle_machine = PrincipalLifecycleStateMachine()

if principal_data.lifecycle_status:
    lifecycle_machine.assert_transition_allowed(
        principal_kind=principal.principal_kind,
        from_status=principal.lifecycle_status,
        to_status=principal_data.lifecycle_status,
        attestation_status=principal.attestation_status,
    )
    # Then update...
```

---

### ⚠️ **#2: Rate Limiter Fail-Open vs Fail-Closed (MEDIUM RISK)**

**OSS Behavior:**
```python
# rate_limiter.py line 145
except RateLimitExceededError:
    raise
except Exception as e:
    logger.critical("Rate limiter unavailable — denying request (fail-closed)")
    raise RateLimitExceededError(f"Rate limit check failed (fail-closed): {e}")
```

**Enterprise Behavior:**
```python
# rate_limit.py line 130
except Exception as e:
    print(f"Rate limit error: {e}")
    return await call_next(request)  # ← FAIL-OPEN
```

**Risk:**
- Enterprise allows **unlimited requests** when Redis is down
- Exposes Enterprise API to brute-force/DoS during Redis outages
- Violates Caracal fail-closed security posture

**Fix:**
```python
except Exception as e:
    logger.critical(f"Rate limit check failed (fail-closed): {e}")
    return Response(
        content='{"error": {"code": "RATE_LIMIT_UNAVAILABLE", "message": "Rate limiting service unavailable"}}',
        media_type="application/json",
        status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
    )
```

---

### ⚠️ **#3: Missing Attestation Guard in Enterprise Routes (MEDIUM RISK)**

**OSS Behavior:**
- [authority.py](../Caracal/packages/caracal-server/caracal/core/authority.py#L66) `AuthorityReasonCode.PRINCIPAL_NOT_ATTESTED`
- Authority evaluator denies mandates if principal is not attested

**Enterprise Behavior:**
- No attestation check in [principals.py](../caracalEnterprise/services/api/src/caracal_api/routes/principals.py)
- Enterprise allows unattested principals to be activated/modified

**Risk:**
- Unattested principals may receive mandates via API before attestation completes
- Creates window where trust boundary is incomplete

**Fix:**
Add attestation check before lifecycle transitions:
```python
if principal.attestation_status != PrincipalAttestationStatus.ATTESTED.value:
    if target_status == PrincipalLifecycleStatus.ACTIVE.value:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail={"error": {"code": "ATTESTATION_REQUIRED", "message": "Principal must be attested before activation"}}
        )
```

---

## INTENTIONAL DIFFERENCES (No Fix Required)

### ✅ **Enterprise Adds Workspace Isolation Layer**
Enterprise's `EnterpriseCaracalClient` adds `Principal.owner == self.workspace_id` to all queries. This is **stronger** than OSS (which is single-tenant). No OSS changes needed.

### ✅ **Enterprise Adds Privileged Re-Auth**
Enterprise API has step-up auth for sensitive operations (e.g., key rotation, billing). OSS CLI/TUI don't need this (already privileged local sessions).

### ⚠️ **Error Response Format Divergence**
OSS uses Python exceptions, Enterprise uses HTTP JSON errors. **Document the mapping** in integration docs but keep separate.

---

## RECOMMENDATIONS

### Immediate (High Priority)
1. **Import and enforce `PrincipalLifecycleStateMachine` in Enterprise routes** before any lifecycle mutations
2. **Change Enterprise rate limiter to fail-closed** on Redis errors

### Medium Priority
3. Add attestation status checks in Enterprise principal activation flows
4. Document rate limit threshold differences (OSS: per-mandate, Enterprise: per-endpoint)

### Low Priority
5. Create error code mapping doc between OSS exceptions and Enterprise HTTP codes
6. Consider harmonizing audit event metadata structure (currently compatible but undocumented)

---

## SUMMARY

**Total Controls Evaluated:** 12  
**Aligned:** 7  
**Critical Drift:** 1  
**Medium Drift:** 2  
**Intentional Differences:** 2

Enterprise correctly delegates to OSS for **JWT validation, authority decisions, session deny-lists, vault enforcement, and audit logging**. The main gaps are **lifecycle state machine bypass** and **fail-open rate limiting**, both fixable by importing OSS components already present in the codebase.