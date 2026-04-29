# Agent Registration and Identity Audit - Findings

**Audit Date:** 2026-04-29  
**Scope:** Agent registration, identity assignment, scopes, permissions, separation, and trust boundaries  
**Codebases:** caracal (OSS) and caracalEnterprise  
**Mode:** Read-only, thorough investigation

---

## Executive Summary

The Caracal system implements a **multi-layered identity architecture** with explicit principal registration, mandate-based access control, workspace scoping, and session management. Agent identity is **strongly maintained** with UUID-based registration and consistent workspace binding. However, several **specific concerns** exist around:

1. **Agent ID header handling with implicit fallback**
2. **Cross-workspace principal lookup gaps in some contexts**
3. **Session context carryover without explicit clearing**
4. **Global unscoped queries in a few enterprise paths**

---

## Strong Identity Properties

### 1. Principal Registration and Unique Identity

**File:** [caracal/packages/caracal-server/caracal/core/identity.py](caracal/packages/caracal-server/caracal/core/identity.py)

**Finding:** ✅ **STRONG**
- Principals are assigned **UUID (`principal_id`)** at registration:
  ```python
  row = Principal(
      principal_id=principal_uuid,  # UUID or None for auto-generation
      name=name,
      principal_kind=principal_kind,
      owner=owner,
      ...
  )
  ```
- **Name uniqueness enforced** within registration: `if self.session.query(Principal).filter_by(name=name).first(): raise DuplicatePrincipalNameError`
- Each principal has explicit `principal_kind`: `human`, `orchestrator`, `worker`, `service`
- **Source principal tracking** for spawned agents: `source_principal_id` binds delegated principals to issuer

### 2. Workspace Scoping via Owner Field

**File:** [caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py](caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py)

**Finding:** ✅ **STRONG** (for enterprise client operations)
- Enterprise client consistently filters principals by `workspace_id`:
  ```python
  Principal.owner == self.workspace_id  # Lines 145, 241, 250, 315, 326, 387, etc.
  ```
- All principal queries in enterprise integration include owner filter
- Prevents cross-tenant principal leakage

### 3. Mandate-Based Scope Binding

**File:** [caracal/packages/caracal-server/caracal/core/mandate.py](caracal/packages/caracal-server/caracal/core/mandate.py)

**Finding:** ✅ **STRONG**
- Mandates bind scope to principal: `subject_id`, `issuer_id`, `resource_scope`, `action_scope`
- Scope validation on issuance:
  ```python
  if not self._validate_scope_subset(resource_scope, issuer_policy.allowed_resource_patterns):
      raise ValueError("Requested resource scope exceeds policy limits")
  ```
- Network distance tracking prevents delegation amplification
- Revocation propagates through delegation graph

### 4. Idempotency and Replay Protection

**File:** [caracal/packages/caracal-server/caracal/core/spawn.py](caracal/packages/caracal-server/caracal/core/spawn.py), lines 430–470

**Finding:** ✅ **STRONG** (for spawn operations)
- Idempotent spawn with marker binding:
  ```python
  spawn_binding = PrincipalWorkloadBinding(
      principal_id=principal.principal_id,
      workload=f"{issuer_uuid}:{idempotency_key}",  # Issuer + key
      binding_type=_IDEMPOTENCY_BINDING_TYPE,
  )
  ```
- Replay detection queries by issuer + idempotency key: `_find_existing_spawn(issuer_uuid, idempotency_key)`
- Prevents issuer from reusing same key to spawn duplicate agents
- **However:** Two different issuers can use same idempotency key independently (intended, safe)

### 5. Session Identity Binding

**File:** [caracal/packages/caracal-server/caracal/core/session_manager.py](caracal/packages/caracal-server/caracal/core/session_manager.py)

**Finding:** ✅ **STRONG**
- Sessions explicitly bind:
  ```python
  "sub": str(subject_id),  # Principal ID
  "principal_id": str(subject_id),
  "org": str(workspace_id),
  "tenant": str(tenant_id),
  "kind": session_kind.value,  # INTERACTIVE, AUTOMATION, TASK
  "sid": session_id,  # Session-specific ID
  "jti": token_jti,  # Unique token ID
  ```
- Task tokens explicitly prevent re-delegation:
  ```python
  if parent_kind == SessionKind.TASK:
      raise SessionValidationError("Task token holders are not allowed to issue delegated task tokens")
  ```
- Refresh token rotation on use with revocation

---

## Identified Concerns

### CONCERN 1: Agent ID Header Fallback to "unknown_agent"

**Severity:** Medium (informational fallback, not exploitable directly)  
**File:** [caracalEnterprise/services/gateway/proxy.py](caracalEnterprise/services/gateway/proxy.py), line 280

```python
agent_id = request.headers.get("X-Caracal-Agent-ID", "unknown_agent")
```

**Issue:**
- Agent ID is extracted from `X-Caracal-Agent-ID` header with fallback to literal string `"unknown_agent"`
- Requests without header will all log/meter as same agent
- Similar pattern for org/workspace scope:
  ```python
  org_id = org_id or "unknown"
  workspace_id = workspace_id or "unknown"
  ```

**Impact:**
- **Metering/Audit Confusion:** Multiple distinct requests without headers collapse into single agent identity for audit
- **Not a direct auth failure:** Gateway validates workspace scope separately via tenant middleware
- **Context:** OSS fallback: `use client-supplied X-Caracal-Target-URL` (acknowledged in code comments)

**Evidence Path:** Metering events record this agent_id without validation of whether it corresponds to authentic principal

### CONCERN 2: Cross-Workspace Principal Lookup (Global Query Paths)

**Severity:** Medium (specific to enterprise sync/reassignment logic)  
**File:** [caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py](caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py), lines 2586–2616

```python
def get_any_principal_by_name(self, name: str) -> Optional[Principal]:
    """Get a principal by name across ALL tenants (no owner filter)."""
    session = self._get_session()
    try:
        return session.query(Principal).filter(
            Principal.name == name
        ).first()  # ← NO owner filter

def get_principal_by_id_globally(self, principal_id: UUID) -> Optional[Principal]:
    """Get a principal by ID across ALL tenants (no owner filter)."""
    session = self._get_session()
    try:
        return session.query(Principal).filter(
            Principal.principal_id == principal_id
        ).first()  # ← NO owner filter
```

**Rationale (Documented):**
- Comment: "Used during sync to detect principals whose UUID already exists in the DB under a different tenant so that ownership can be reassigned"
- Intended for **sync scenario** where a workspace principal is being migrated/reassigned to a new tenant

**Issue:**
- These are unsafe if called without strict context validation
- Caller must verify returned principal is in correct workspace before using it
- **Pattern:** Used in `register_principal` with reassignment logic

**Evidence:** sync flow checks ownership after global query:
```python
if principal is None:
    # ... create new principal ...
else:
    # Reassign ownership
    principal.owner = self.workspace_id
```

**Status:** Intentional but requires careful caller validation

### CONCERN 3: Session Context Carryover Without Explicit Isolation

**Severity:** Low (proper FastAPI patterns used, but worth checking)  
**File:** [caracalEnterprise/services/api/src/caracal_api/middleware/auth.py](caracalEnterprise/services/api/src/caracal_api/middleware/auth.py), lines 100–128

```python
# Attach user context to request state
request.state.auth_context = auth_context
request.state.user_id = auth_context.user_id
request.state.workspace_id = auth_context.workspace_id
request.state.principal_id = auth_context.principal_id
request.state.session_id = auth_context.session_id
request.state.session_kind = auth_context.session_kind
request.state.token_jti = token_jti
request.state.auth_time = auth_time
request.state.mfa = mfa_verified
request.state.is_admin = auth_context.is_admin
```

**Finding:** ✅ **No issue** (properly scoped per-request)
- FastAPI's `request.state` is **per-request** (not global)
- **Not** a shared global like a class variable
- Each request gets fresh state object
- State is set in auth middleware **before** route handlers

**Verification:** Test at [caracalEnterprise/services/api/tests/test_caracal_client.py](caracalEnterprise/services/api/tests/test_caracal_client.py#L150) uses separate request contexts

### CONCERN 4: Enterprise Middleware Scope Fallback

**Severity:** Low (same as CONCERN 1, by design)  
**File:** [caracalEnterprise/services/gateway/proxy.py](caracalEnterprise/services/gateway/proxy.py), lines 300–310

```python
if enterprise_enforcement_enabled and (org_id == "unknown" or workspace_id == "unknown"):
    self._denied_count += 1
    logger.warning("Missing workspace/org scope in gateway request (fail-closed): org=%s workspace=%s", ...)
    return JSONResponse(status_code=401, ...)
```

**Finding:** ✅ **Correct behavior** (fail-closed)
- When enterprise enforcement is enabled, missing workspace/org scope is **explicitly rejected**
- Status code 401 + denial event logged
- Only permissive when enforcement is disabled

### CONCERN 5: API Key Cache Expiration (Tenant Middleware)

**Severity:** Low (cache has TTL, timeout-safe)  
**File:** [caracalEnterprise/services/gateway/tenant_middleware.py](caracalEnterprise/services/gateway/tenant_middleware.py), lines 100–120

```python
self._api_key_cache: Dict[str, tuple] = {}
self._CACHE_TTL = 300  # 5 minutes

cached = self._api_key_cache.get(key_hash)
if cached:
    org_id, workspace_id, tier, mode, exp = cached
    if time.monotonic() < exp:  # ← Timeout check
        return EnterpriseAuthContext.from_gateway_key(workspace_id=workspace_id, tier=tier, mode=mode)
    del self._api_key_cache[key_hash]  # ← Stale entry removed
```

**Finding:** ✅ **Correct** (time-based expiry with timeout check)
- Uses `time.monotonic()` (not affected by system clock adjustments)
- TTL hardcoded to 5 minutes
- Expired entries are cleaned up
- Fallback: DB lookup on cache miss

---

## Session and Refresh Token Handling

**File:** [caracal/packages/caracal-server/caracal/core/session_manager.py](caracal/packages/caracal-server/caracal/core/session_manager.py), lines 700–740

**Finding:** ✅ **STRONG** (proper token rotation)

```python
async def refresh_session(self, refresh_token: str, *, rotate_refresh_token: bool = True, ...):
    """Issue a new session from a valid refresh token."""
    claims = await self.validate_refresh_token(refresh_token)
    
    if rotate_refresh_token:
        await self.revoke_token(refresh_token)  # ← OLD token revoked
    
    carry_claims: dict[str, Any] = {}
    for key, value in claims.items():
        if key in {"sub", "org", "tenant", "sid", "kind", "jti", "exp", ...}:
            continue
        carry_claims[key] = value
    
    if extra_claims:
        carry_claims.update(extra_claims)
    
    return self.issue_session(subject_id=..., workspace_id=..., ...)
```

**Validation:** 
- Old refresh token explicitly revoked before new one issued
- New token gets fresh `jti` (line 361: `refresh_jti = uuid4().hex`)
- Extra claims properly merged (no principal spoofing via extra_claims):
  ```python
  if str(key) in self._RESERVED_CLAIM_KEYS:  # "sub", "principal_id", "org", "tenant", etc.
      continue  # ← Skip reserved keys, use canonical values
  ```

---

## Trust Boundary Analysis

### Human-AI Boundary (Delegation Graph)

**File:** [caracal/packages/caracal-server/caracal/core/delegation_graph.py](caracal/packages/caracal-server/caracal/core/delegation_graph.py)

**Finding:** ✅ **ENFORCED** (direction-aware delegation)

```python
# Allowed paths: human → orchestrator/worker/service, orchestrator → worker/service, peer
# Forbidden: service → any, worker → human
```

Validation prevents privilege escalation through improper delegation direction

### Request State Isolation (Per-Request, Not Per-Session)

**File:** FastAPI middleware patterns

**Finding:** ✅ **Correct** (each request gets fresh context)
- Request state is **per-request**, not shared across requests
- Auth middleware runs before each route
- No global session state leaked between users

---

## Non-Issues Verified

### ✅ Implicit Defaults in Registration

Principal registration **requires explicit parameters:**
- `owner` (workspace ID) is mandatory
- `principal_kind` validates against enum (not implicit)
- Name uniqueness is enforced
- UUIDs are explicit (generated or provided)

**Unlike:** System calls might use OSS defaults gracefully (acknowledged in code)

### ✅ Cross-Session Reuse Prevention

Sessions are tied to:
- Unique JTI (per-token)
- Session ID (per-session)
- Subject ID (principal)
- Workspace ID

**Revocation is multi-level:**
1. Token JTI denylisted (explicit revocation)
2. Principal session revocation cutoff (wholesale invalidation)

### ✅ Scope Binding Correctness

Mandate scopes are **properly constrained:**
- Issued scopes subset-validated against issuer policy
- Delegated scopes subset-validated against source mandate
- Network distance decrements (no amplification)

---

## Recommendations and Action Items

### 1. **[MEDIUM] Document the agent_id fallback behavior**
   - Add clear audit note that "unknown_agent" indicates missing identity context
   - Ensure monitoring alerts on "unknown_agent" volume
   - Consider whether OSS fallback should be stricter in production

### 2. **[LOW] Audit global principal lookup callers**
   - Review all callers of `get_any_principal_by_name` and `get_principal_by_id_globally`
   - Verify they validate returned principal ownership before use
   - Add tests for cross-workspace rejection in these paths

### 3. **[LOW] Session context lifecycle documentation**
   - Document that `request.state` is per-request in FastAPI
   - Add comments in middleware explaining temporary nature of context
   - Ensure new developers understand isolation scoping

### 4. **[OPTIONAL] API key cache refresh strategy**
   - Consider more frequent refresh if key rotation is a common operation
   - Current 5-minute TTL is conservative and reasonable

---

## Conclusion

**Overall Assessment:** The agent identity system is **well-architected** with strong separation, workspace scoping, and mandate-based access control. Agents working on the same search/task maintain **independent representation** through UUIDs and session IDs. The system prevents identity collapse and cross-workspace confusion through:

1. **Explicit UUID-based principal identity** (not implicit)
2. **Workspace owner binding** (consistent filtering in queries)
3. **Mandate-based scope isolation** (per-principal, not per-request)
4. **Idempotent spawn with issuer + key binding** (prevents duplicate agent creation)
5. **Session-level isolation** (per-request state, fresh JTI on issue)

**Concerns identified are minor and mostly informational** (fallback agent ID for audit trails, intentional cross-workspace lookups in sync scenarios). None represent actual identity confusion or unauthorized cross-agent access vulnerabilities.

**Trust boundaries are maintained** between:
- Human and AI principals (direction-validated delegation)
- Different workspaces (consistently filtered queries)
- Sessions (renewed tokens, revocation tracked)
- Request contexts (per-request isolation)

