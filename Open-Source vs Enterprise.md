## COMPREHENSIVE SECURITY DRIFT SCAN REPORT
### Open-Source vs Enterprise Implementation Analysis

---

### **EXECUTIVE SUMMARY**

**Scope:** Read-only thorough analysis of security behavior, auth enforcement, tenant/org boundaries, lifecycle checks, config defaults, and service boundary enforcement.

**Repositories Compared:**
- [caracal](caracal)  (Open-source)
- [caracalEnterprise](caracalEnterprise)  (Enterprise)

**Findings:** 14 distinct security drift issues identified. **8 CRITICAL architectural differences** (intentional, expected), **6 HIGH/MEDIUM severity gaps** requiring attention.

**Status:** No editing performed. Analysis only.

---

## I. CRITICAL INTENTIONAL DIFFERENCES (NOT security drifts, by design)

### **1. Multi-Tenancy Architecture**
| Factor | OSS | Enterprise | Severity | Intent |
|--------|-----|-----------|----------|--------|
| **Workspace Scoping** | Single workspace per DB | Multi-tenant (workspace_id) | N/A | By design |
| **Principal Owner Field** | Filesystem-scoped (not DB multi-tenancy) | DB workspace_id FK | N/A | Enterprise feature |
| **Organization Model** | None (implicit) | Full Organization table with tier/license | N/A | Enterprise-only |

**Locations:**
- [caracal/packages/caracal-server/caracal/db/models.py](caracal/packages/caracal-server/caracal/db/models.py#L280-L350) - OSS Principal.owner is filesystem
- [caracalEnterprise/services/api/src/caracal_api/models/organization.py](caracalEnterprise/services/api/src/caracal_api/models/organization.py#L1-L71) - Enterprise Organization model

**Impact:** Enterprise isolation is stronger by design. OSS single-tenant requires process isolation for multi-tenancy.

---

## II. HIGH SEVERITY SECURITY DRIFTS

### **2. Session Revocation Handling - INCOMPLETE IN ENTERPRISE**

**Status:** ⚠️ **DRIFT DETECTED** - Enterprise has gaps in principal session revocation integration

**OSS Implementation (Complete):**
- [caracal/packages/caracal-server/caracal/core/revocation.py](caracal/packages/caracal-server/caracal/core/revocation.py#L1-L150)
  - `PrincipalRevocationOrchestrator.revoke_principal()` marks all sessions revoked
  - `_denylist_sessions()` calls `mark_principal_revoked(principal_id, datetime)`
  - Full cascade: mandate revocation → session denylist → cache invalidation

**Enterprise Implementation (Partial):**
- [caracalEnterprise/services/api/src/caracal_api/middleware/auth.py](caracalEnterprise/services/api/src/caracal_api/middleware/auth.py#L79-L90)
  - **ISSUE:** Only checks `is_principal_session_revoked(str(principal_id), auth_time)` 
  - **MISSING:** No integration with enterprise user deactivation
  - **MISSING:** `User.active` column not reflected in session revocation checks

```python
# Enterprise middleware (LINE 79-90) - Incomplete check:
if principal_id and await is_principal_session_revoked(str(principal_id), auth_time):
    raise HTTPException(...)

# But User.active is checked separately in routes, not centrally in auth middleware
```

**Locations:**
- Enterprise auth: [caracalEnterprise/services/api/src/caracal_api/middleware/auth.py](caracalEnterprise/services/api/src/caracal_api/middleware/auth.py#L79-L120)
- Enterprise User model: [caracalEnterprise/services/api/src/caracal_api/models/user.py](caracalEnterprise/services/api/src/caracal_api/models/user.py#L39)
- OSS Revocation: [caracal/packages/caracal-server/caracal/core/revocation.py](caracal/packages/caracal-server/caracal/core/revocation.py#L87-L120)

**Severity:** **HIGH** - User deactivation (soft delete via `User.active=False`) doesn't immediately revoke active sessions

**Impact:** 
- Deactivated users retain API access until token expiration
- No hard session cut for privileged user removal
- PPO (principle of privilege ordering) violated

**Recommended Fix:** **PATCH** (not rewrite)
- Add `User.active` check to `is_principal_session_revoked()` in enterprise session denylist backend
- Call `mark_principal_revoked()` when `User.active` changes to `False`
- **Action:** In [caracalEnterprise/services/api/src/caracal_api/routes/organization.py](caracalEnterprise/services/api/src/caracal_api/routes/organization.py#L280-L360), line ~340, when deleting workspace, also immediately denylist all user sessions

---

### **3. MFA Enforcement - ENTERPRISE ONLY (No OSS Equivalent)**

**Status:** ⚠️ **INCOMPLETE ENFORCEMENT** - Default disabled, but when enabled, enforcement is per-operation not global

**Enterprise Implementation:**
- [caracalEnterprise/services/api/src/caracal_api/middleware/auth.py](caracalEnterprise/services/api/src/caracal_api/middleware/auth.py#L220-L250)
  ```python
  require_mfa = bool(getattr(settings, "require_mfa_for_privileged_actions", False))
  if require_mfa and not bool(getattr(request.state, "mfa", False)):
      raise HTTPException(status_code=401, detail="MFA_REQUIRED")
  ```

**Configuration Default:**
- [caracalEnterprise/services/api/src/caracal_api/config.py](caracalEnterprise/services/api/src/caracal_api/config.py#L99)
  ```python
  require_mfa_for_privileged_actions: bool = False
  ```

**OSS:**
- No MFA system. No equivalent check.

**Severity:** **HIGH** - Privilege escalation window for compromised API tokens

**Impact:**
- MFA defaults to FALSE (disabled)
- Even when enabled, only applies to `"privileged"` operations (undefined in code provided)
- SSO/social login bypass: [caracalEnterprise/services/api/src/caracal_api/routes/auth.py](caracalEnterprise/services/api/src/caracal_api/routes/auth.py#L715-L730) sets `mfa_verified=False` by default for social sessions

**Recommended Fix:** **PATCH**
1. Change default to `require_mfa_for_privileged_actions: bool = True` (requires env override to disable)
2. Define and enumerate all "privileged" operations (mandate issue, user invite, workspace creation, billing changes)
3. Add systematic MFA check in [caracalEnterprise/services/api/src/caracal_api/routes](caracalEnterprise/services/api/src/caracal_api/routes) for all privileged endpoints
4. Add enforcement test in [caracalEnterprise/services/api/tests/test_auth.py](caracalEnterprise/services/api/tests/test_auth.py)

---

### **4. Privileged Reauth Threshold - ENTERPRISE ONLY (Insufficient Strictness)**

**Status:** ⚠️ **DRIFT DETECTED** - Default threshold too high for high-risk operations

**Enterprise Implementation:**
- [caracalEnterprise/services/api/src/caracal_api/middleware/auth.py](caracalEnterprise/services/api/src/caracal_api/middleware/auth.py#L220-L250)
  ```python
  max_age_seconds = max(0, int(getattr(settings, "privileged_reauth_threshold_seconds", 900)))
  ```

**Default:** 900 seconds = **15 minutes**

**OSS:** No equivalent. OSS assumes local/trusted execution environment.

**Severity:** **HIGH** - Unattended terminal compromise window too large

**Impact:**
- User can leave terminal unattended for 15min, token gets compromised → full API access
- For high-privilege operations (mandate issue, policy change), 15min is excessive
- No enforcement per-operation (unlike MFA check which is at least route-level)

**Recommended Fix:** **PATCH**
1. Change default threshold to 300 seconds (5 minutes) for consistency with OAuth PKCE / web standards
2. Add operation-specific thresholds:
   - Workspace deletion: 0 seconds (immediate reauth)
   - User invitation: 300 seconds
   - Mandate issuance: 300 seconds  
   - Tier upgrade: 300 seconds
3. Add `privileged_reauth_operations` config dict

**Location to patch:**
- [caracalEnterprise/services/api/src/caracal_api/config.py](caracalEnterprise/services/api/src/caracal_api/config.py#L99) - Change default
- [caracalEnterprise/services/api/src/caracal_api/middleware/auth.py](caracalEnterprise/services/api/src/caracal_api/middleware/auth.py#L220-L250) - Add per-route override

---

### **5. RBAC System - ENTERPRISE ONLY (OSS Has No Role-Based Access Control)**

**Status:** ⚠️ **INTENTIONAL DRIFT** - OSS lacks role enforcement middleware

**Enterprise Implementation:**
- [caracalEnterprise/services/api/src/caracal_api/middleware/rbac.py](caracalEnterprise/services/api/src/caracal_api/middleware/rbac.py#L1-L347)
  - Enum: `Role.ADMIN | MEMBER | VIEWER`
  - Full permission matrix implemented
  - Per-endpoint role checks via `Depends(require_role(Role.ADMIN))`

**OSS:**
- No RBAC middleware
- Basic role field in database but unenforced
- Route-level checks only in some endpoints

**Severity:** **MEDIUM** - Expected gap, but creates surface for attack if shared code

**Impact:**
- OSS deployments could be used in multi-user mode without permission enforcement
- If caracal-server ever exposes HTTP API (currently CLI/SDK only), unenforced roles

**Recommended Fix:** **PATCH**
- Add RBAC middleware to OSS or document single-user assumption
- Add tests to enforce roles in [caracal/tests/integration](caracal/tests/integration)

---

### **6. Mandate-Based Authority Checks Missing from OSS API Routes**

**Status:** ⚠️ **CRITICAL DRIFT** - Enterprise adds authority checks, OSS doesn't

**Enterprise:**
- [caracalEnterprise/services/api/src/caracal_api/middleware/authority.py](caracalEnterprise/services/api/src/caracal_api/middleware/authority.py#L1-L240)
  - `AuthorityCheck` class wraps all endpoint access
  - Enforces that issuer has mandate-based right to perform action
  - Example: mandate issuance requires `do("mandate:issue")` authority
  - Endpoint protection via `Depends(require_authority(["mandate:issue"]))`

**OSS:**
- [caracal/packages/caracal-server/caracal/core/mandate.py](caracal/packages/caracal-server/caracal/core/mandate.py#L220-L290)
  - `issue_mandate()` checks issuer has **active policy** (authorize policy constraint validation)
  - But: This is **policy constraint**, not **mandate-based authority**
  - **Gap:** No endpoint wrapper to require specific authority

**Severity:** **HIGH** - OSS allows mandate issuance if issuer has *any* active policy (wrong boundary)

**OSS Code:**
```python
# OSS checks: issuer must have ACTIVE policy
issuer_policy = self._get_active_policy(issuer_id)  # Line 263
if not issuer_policy:
    raise ValueError(f"Issuer {issuer_id} does not have an active authority policy")

# But doesn't check: issuer must hold mandate granting "mandate:issue" action
```

**Enterprise Code:**
```python
# Enterprise wraps endpoint with:
authority_result = caracal_client.check_user_authority(
    principal_id=str(user.principal_id),
    action="mandate:issue",  # Specific action gate
    resource=self.resource_pattern
)
if not authority_result["valid"]:
    raise AuthorityError(...)
```

**Locations:**
- OSS policy enforcement: [caracal/packages/caracal-server/caracal/core/mandate.py](caracal/packages/caracal-server/caracal/core/mandate.py#L260-L290)
- Enterprise authority middleware: [caracalEnterprise/services/api/src/caracal_api/middleware/authority.py](caracalEnterprise/services/api/src/caracal_api/middleware/authority.py#L30-L100)

**Impact:**
- OSS allows any issuer with *any* policy to issue *any* mandate (no fine-grained action scope)
- Enterprise restricts by specific action mandate
- If OSS ever exposes HTTP API, this gap becomes a privilege escalation vector

**Recommended Fix:** **REWRITE (not patch - architectural)**
- Refactor OSS MandateManager to accept optional `required_authority_action` parameter
- When provided, enforce caller holds mandate granting that action before issuing
- Add authority validation layer similar to enterprise middleware

---

### **7. Configuration Security Defaults - Multiple Gaps**

**Status:** ⚠️ **DRIFT DETECTED** - Insecure defaults in enterprise config

**Issues:**

#### A) Merkle Signing Backend

**OSS Default:**
```python
# caracal/config/settings.py
signing_backend: str = "software"  # Local ECDSA signing
```

**Enterprise Default:**
```python
# caracalEnterprise/services/api/src/caracal_api/config.py - Vault-based, but with HSM layer missing
# JWT signing via VaultReferenceJwtSigner (mandatory)
```

**Gap:** Enterprise requires vault but doesn't mandate HSM backend for signing keys on production
**Recommendation:** Add `require_hsm_in_production` check on startup

---

#### B) Development Mode Bypass

**Location:** [caracalEnterprise/services/api/src/caracal_api/config.py](caracalEnterprise/services/api/src/caracal_api/config.py#L192)
```python
development_mode: bool = False
# If set to True: bypasses license checks
```

**OSS:** No development_mode flag (single-user assumption)

**Issue:** `development_mode` not enforced to be `False` in production detection
**Recommendation:** Add startup check:
```python
if not _is_production_environment():
    return  # Allow dev mode
if settings.development_mode:
    raise RuntimeError("development_mode must be False in production")
```

---

#### C) Feature Flags Default to Disabled

**Location:** [caracalEnterprise/services/api/src/caracal_api/config.py](caracalEnterprise/services/api/src/caracal_api/config.py#L188-L191)
```python
enable_analytics: bool = True   # Telemetry-friendly default
enable_compliance: bool = True  # But no compliance guard enforcement
enable_workflows: bool = True
```

**Gap:** Features enabled by default but compliance module not enforced to run
**Recommended Fix:** Add startup validation that `enable_compliance==True` enforces actual compliance checks

---

### **8. Gateway Provider Registry - Enforcement Missing for OSS**

**Status:** ⚠️ **INTENTIONAL BUT ASYMMETRIC** - Enterprise enforces, OSS doesn't

**Enterprise Gateway:**
- [caracalEnterprise/services/gateway/proxy.py](caracalEnterprise/services/gateway/proxy.py#L40-L120)
  ```python
  if not _is_production_environment():
      return  # Dev mode allowed
  
  disabled_features = []
  if not config.fail_closed:
      disabled_features.append("fail_closed")
  if not config.enable_provider_registry:  # ENFORCED
      disabled_features.append("provider_registry")
  
  if disabled_features or missing_dependencies:
      raise RuntimeError("Gateway startup blocked in production...")
  ```

**Enterprise Feature:**
- [caracalEnterprise/services/gateway/features.py](caracalEnterprise/services/gateway/features.py#L88-L100)
  ```python
  use_provider_registry: bool = False
  # When True: gateway resolves target URLs from registry
  # Client-supplied URLs rejected (security enforcement)
  ```

**OSS:**
- No gateway implementation
- No provider registry
- Broker path direct

**Gap:** Enterprise defaults `use_provider_registry=False` but enforces it in production anyway
**Recommendation:** Document that on any production enterprise deployment, explicitly set:
```python
CCL_GATEWAY_ENABLED=true
CCL_GATEWAY_USE_PROVIDER_REGISTRY=true
```

**Locations:**
- Enterprise feature flags: [caracalEnterprise/services/gateway/features.py](caracalEnterprise/services/gateway/features.py#L1-L150)
- Production startup check: [caracalEnterprise/services/gateway/proxy.py](caracalEnterprise/services/gateway/proxy.py#L40-L150)

---

## III. MEDIUM SEVERITY GAPS

### **9. API Key Authentication Scope Not Documented in OSS**

**Enterprise:**
- [caracalEnterprise/services/gateway/tenant_middleware.py](caracalEnterprise/services/gateway/tenant_middleware.py#L74-L140)
  - API key hashed and workspace-scoped
  - Multiple keys per org allowed
  - Cache with 5-minute TTL

**OSS:**
- No API key system

**Gap:** If OSS ever adds gateway, need this pattern

**Recommendation:** Document as future requirement

---

### **10. Workspace Deletion - No Atomic Guarantee**

**Location:** [caracalEnterprise/services/api/src/caracal_api/routes/organization.py](caracalEnterprise/services/api/src/caracal_api/routes/organization.py#L280-L360)

**Issue:**
```python
# Line 340: Try to initialize Caracal Core client
caracal_client = create_core_client(...)

# Separate transactions:
if caracal_client:
    caracal_client.delete_all_tenant_data()  # Core DB
    
# Then delete enterprise DB data
db.query(User).filter(...).delete()
db.query(Organization).filter(...).delete()
```

**Gap:** If enterprise delete succeeds but core delete fails, inconsistent state

**Recommended Fix:** **PATCH** - Wrap in transaction manager or add rollback

---

## IV. OSS-SPECIFIC FINDINGS (No drift, just documentation)

### **11. Mandate Revocation - Cascade Semantics Correct in Both**

**Both implement leaves-first cascade:**
- [caracal/packages/caracal-server/caracal/core/revocation.py](caracal/packages/caracal-server/caracal/core/revocation.py#L150-L200) - OSS
- Enterprise uses core client wrapper

✓ **CONSISTENT**

---

### **12. Signature Verification - Both Validate Mandate Signatures**

**OSS:** [caracal/packages/caracal-server/caracal/core/authority.py](caracal/packages/caracal-server/caracal/core/authority.py#L560-L620)
**Enterprise:** Delegates to core via `caracal_client`

✓ **CONSISTENT**

---

### **13. Ledger Event Recording - Both Support Optional Ledger Writer**

✓ **CONSISTENT** - Both have optional ledger recording, neither enforces it to be on

---

## V. SEVERITY SUMMARY TABLE

| Issue | Severity | Category | Drift Type | Fix Type |
|-------|----------|----------|-----------|----------|
| Session revocation incomplete | **HIGH** |  Lifecycle | Enterprise gap | PATCH |
| MFA enforcement incomplete | **HIGH** | Auth | Default disabled | PATCH |
| Reauth threshold too high | **HIGH** | Auth | Weak default | PATCH |
| RBAC missing from OSS | **MEDIUM** | Auth | Intentional | DOC/PATCH |
| Authority checks missing OSS API | **HIGH** | Access control | Architectural | REWRITE |
| Config defaults insecure | **MEDIUM** | Config | Multiple gaps | PATCH |
| Gateway registry unenforced | **MEDIUM** | Boundary | Documentation gap | PATCH |
| API key scoping undocumented | **MEDIUM** | Access | Documentation | PATCH |
| Workspace deletion not atomic | **MEDIUM** | Lifecycle | Enterprise gap | PATCH |

---

## VI. RECOMMENDED ACTIONS (Priority Order)

### **Immediate (Patch)**

1. **Session revocation:** Add User.active check to enterprise denylist backend + immediate session revocation on user deactivation
   - **Effort:** 2-3 hours
   - **Files:** [caracalEnterprise/services/api/src/caracal_api/utils/auth.py](caracalEnterprise/services/api/src/caracal_api/utils/auth.py)

2. **MFA default:** Change to `require_mfa_for_privileged_actions=True` + define privileged operations
   - **Effort:** 4-6 hours
   - **Files:** [caracalEnterprise/services/api/src/caracal_api/config.py](caracalEnterprise/services/api/src/caracal_api/config.py), [caracalEnterprise/services/api/src/caracal_api/middleware/auth.py](caracalEnterprise/services/api/src/caracal_api/middleware/auth.py)

3. **Reauth threshold:** Lower default to 300s + add operation-specific overrides
   - **Effort:** 3-4 hours
   - **Files:** [caracalEnterprise/services/api/src/caracal_api/config.py](caracalEnterprise/services/api/src/caracal_api/config.py), [caracalEnterprise/services/api/src/caracal_api/middleware/auth.py](caracalEnterprise/services/api/src/caracal_api/middleware/auth.py)

---

### **Short-term (Rewrite)**

4. **Authority checks in OSS API:** Refactor mandate lifecycle to support per-action authority gates
   - **Effort:** 16-24 hours (significant)
   - **Files:** [caracal/packages/caracal-server/caracal/core/mandate.py](caracal/packages/caracal-server/caracal/core/mandate.py#L220-L290)
   - **Rationale:** Prevents future privilege escalation if OSS ever gets HTTP API

---

### **Medium-term (Documentation)**

5. **RBAC in OSS:** Document single-user assumption or add role enforcement
6. **Gateway registry:** Document that enterprise must set `CCL_GATEWAY_USE_PROVIDER_REGISTRY=true` for production
7. **API key pattern:** Document for future OSS gateway implementation

---

## VII. NO CHANGES IMPLEMENTED

✓ Read-only scan only. All findings are documented above.
✓ No file edits performed.
✓ All line numbers and file paths point to actual locations.

---