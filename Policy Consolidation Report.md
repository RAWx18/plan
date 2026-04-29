## Policy Consolidation Report: Hardcut/Preflight/Policy Sprawl Analysis

### Executive Summary

Comprehensive scan of `/home/raw/Documents/workspace/caracalEcosystem/caracal` and `/home/raw/Documents/workspace/caracalEcosystem/caracalEnterprise` reveals **moderate-to-high policy sprawl complexity** across three key layers:

1. **Hardcut/Preflight validation** (centralized but incomplete)
2. **Authorization enforcement** (scattered across multiple handlers)
3. **Policy gate constraints** (duplicated across CLI, MCP, and entrypoints)

**Critical finding: Authorization logic bypassed by incomplete separation of concerns** — constraint enforcement mixed with business logic in handler code.

---

## Detailed Findings

### 1. HARDCUT/PREFLIGHT SCOPE FRAGMENTATION

**Severity**: HIGH | **Impact**: HIGH | **Complexity**: Complex

#### Issues

| File | Problem | Severity |
|------|---------|----------|
| [caracal/packages/caracal/caracal/runtime/hardcut_preflight.py](caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L1-L535) | Monolithic file (535 lines) with 40+ hardcoded marker constants; difficult to maintain and extend | MODERATE |
| [caracal/packages/caracal/caracal/runtime/hardcut_preflight.py](caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L20) | `_FORBIDDEN_SYNC_RUNTIME_MODEL_MARKERS` - hardcoded tuples; no centralized registry | MODERATE |
| [caracal/packages/caracal/caracal/runtime/hardcut_preflight.py](caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L82-L116) | `_REQUIRED_SECRET_BACKEND_ENV`, `_ALLOWED_SECRET_BACKENDS` scattered as module-level constants instead of configuration | HIGH |
| [caracalEnterprise/services/api/src/caracal_api/main.py](caracalEnterprise/services/api/src/caracal_api/main.py#L70) | `_assert_registration_state_hardcut_contract()` — separate enterprise hardcut contract; **duplicates** hardcut concepts | HIGH |

#### Specific Rule Sprawl

**Module-level constant hardcut rules (caracal/packages/caracal/caracal/runtime/hardcut_preflight.py)**:
```
Lines 22-116:
- _CANONICAL_ENTERPRISE_API_FAMILY
- _CANONICAL_ENTERPRISE_CLI_FAMILY
- _FORBIDDEN_SYNC_RUNTIME_MODEL_MARKERS
- _FORBIDDEN_SQLITE_PREFIXES
- _FORBIDDEN_RUNTIME_COMPOSE_MARKERS
- _FORBIDDEN_ENTERPRISE_COMPOSE_MARKERS
- _REQUIRED_RUNTIME_COMPOSE_MARKERS
- _REQUIRED_ENTERPRISE_COMPOSE_MARKERS
- _FORBIDDEN_STATE_RELATIVE_PATHS
- _FORBIDDEN_COMPAT_ENV_VARS (7 variants)
- _REQUIRED_SECRET_BACKEND_ENV
- _ALLOWED_SECRET_BACKENDS
- _REQUIRED_VAULT_ENV_VARS (4 variants)
- _SESSION_SIGNING_ALGORITHM_ENV_VARS
- _ALLOWED_SESSION_SIGNING_ALGORITHMS
- _FORBIDDEN_CONFIG_MARKERS (16 variants)
- _GATEWAY_URL_ENV_KEYS
- _SECRET_FILE_NAMES
```

This is not managed through a configuration object—rules are hardcoded file constants.

**Recommendation**: REWRITE to centralized rules registry with versioning.

---

### 2. AUTHORIZATION LOGIC SCATTERED ACROSS HANDLERS

**Severity**: CRITICAL | **Impact**: CRITICAL | **Complexity**: Very High

#### Issues

| File | Function | Problem | Pattern |
|------|----------|---------|---------|
| [caracal/packages/caracal/caracal/runtime/entrypoints.py](caracal/packages/caracal/caracal/runtime/entrypoints.py#L2157) | `_authorize_caller_for_target()` | Authorization check embedded **inside handler builder** (_build_ais_handlers). Defined inline as closure; not reusable; testing requires mocking the entire builder. | BYPASS-PRONE |
| [caracal/packages/caracal/caracal/runtime/entrypoints.py](caracal/packages/caracal/caracal/runtime/entrypoints.py#L2280) | `_authorize_issue_token_request()` | Similar inline closure; **duplicate logic** checking capabilities via `_caller_has_capability()`. | SPRAWL |
| [caracal/packages/caracal/caracal/runtime/entrypoints.py](caracal/packages/caracal/caracal/runtime/entrypoints.py#L2125-L2145) | `_caller_has_capability()` | Hardcoded "system.admin" magic string; checks capabilities both in claims AND in database; **dual-path authority check defeats fail-closed**. | BYPASSABLE |
| [caracal/packages/caracal-server/caracal/mcp/adapter.py](caracal/packages/caracal-server/caracal/mcp/adapter.py#L1725) | `_authorize_principal_request()` | Separate authorization layer in MCP adapter; **different logic** than entrypoints.py handlers. | DIVERGENT |
| [caracal/packages/caracal-server/caracal/mcp/service.py](caracal/packages/caracal-server/caracal/mcp/service.py#L273-L445) | Route handlers (POST /tools/call, etc.) | **Auth checks (401/403) scattered inline** in route handlers, mixed with business logic. Lines 273, 280, 312, 318, 326, 333, 341, 357, 363, 410, 420, 425, 445. | SPRAWL |

#### Critical Pattern: Dual-Path Capability Checking

[entrypoints.py lines 2125-2145](caracal/packages/caracal/caracal/runtime/entrypoints.py#L2125):
```python
def _caller_has_capability(
    self_claims: dict[str, Any] | None,
    caller_principal_id: str,
    *capabilities: str,
) -> bool:
    requested = {str(capability).strip() for capability in capabilities if str(capability).strip()}
    if not requested:
        return False

    # PATH 1: Check JWT claims
    claim_capabilities = {
        str(capability).strip()
        for capability in (self_claims or {}).get("capabilities", []) or []
        if str(capability).strip()
    }
    if "system.admin" in claim_capabilities or requested.intersection(claim_capabilities):
        return True

    # PATH 2: Check database (fallback)
    with resolved_db_manager.session_scope() as session:
        query = getattr(session, "query", None)
        if not callable(query):
            return False
        # ... database lookup
```

**Problem**: Capability checking has two paths (JWT + database). If JWT path fails but database check succeeds, authorization decision changes. This violates **fail-closed semantics** — policy enforcement should be at a single point.

#### Consequence: Bypassable Authorization

- JWT token could have stale capability data
- Database lookup in fallback path could fail silently
- Attacker with capability in database but not in JWT can retry until database lookup succeeds
- No unified policy decision point

**Recommendation**: REWRITE as single authorization boundary using centralized authority.py evaluator.

---

### 3. CONFLICTING HARDCUT CONTRACTS (OSS vs. Enterprise)

**Severity**: HIGH | **Impact**: MEDIUM | **Complexity**: High

#### Issues

| Location | Contract | Redundancy |
|----------|----------|-----------|
| [caracal/packages/caracal/caracal/runtime/hardcut_preflight.py](caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L433) | `assert_runtime_hardcut()` | Runtime-specific checks (compose markers, SQLite, JSONB) |
| [caracal/packages/caracal/caracal/runtime/hardcut_preflight.py](caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L465) | `assert_enterprise_hardcut()` | Enterprise-specific checks (gateway execution mode) |
| [caracalEnterprise/services/api/src/caracal_api/main.py](caracalEnterprise/services/api/src/caracal_api/main.py#L70) | `_assert_registration_state_hardcut_contract()` | **SEPARATE** enterprise-only registration metadata hardcut; **not called from OSS code** |

**Problem**: Enterprise API has its own hardcut contract validation (`_assert_registration_state_hardcut_contract`) that is NOT tested in OSS paths. The two validation systems are decoupled:
- OSS: uses `assert_runtime_hardcut()`, `assert_enterprise_hardcut()`
- Enterprise: adds `_assert_registration_state_hardcut_contract()`

No unified entry/exit point. Enterprise contract violations could be missed if called out of order.

**Recommendation**: CONSOLIDATE to single `assert_full_hardcut_contract()` that chains all checks in defined order.

---

### 4. POLICY & CONSTRAINT VALIDATION SCATTERED

**Severity**: MODERATE | **Impact**: MODERATE | **Complexity**: Moderate

#### Issues

| File | Pattern | Problem |
|------|---------|---------|
| [caracal/packages/caracal-server/caracal/cli/authority_policy.py](caracal/packages/caracal-server/caracal/cli/authority_policy.py#L100-L110) | CLI command validation | Inline validation (UUID parsing, range checks) duplicates what should be in core/authority_policy.py |
| [caracal/packages/caracal-server/caracal/cli/authority_policy.py](caracal/packages/caracal-server/caracal/cli/authority_policy.py#L149) | `validate_provider_scopes()` | Called from CLI; also used in flow/screens (duplication). Should be in core module. |
| [caracal/packages/caracal-server/caracal/core/authority.py](caracal/packages/caracal-server/caracal/core/authority.py#L30-L65) | `AuthorityReasonCode` class | Reason codes defined as class constants; could be extensible enum or registry. |
| [caracal/packages/caracal-server/caracal/mcp/policy_gate.py](caracal/packages/caracal-server/caracal/mcp/policy_gate.py#L1-L10) | PassThrough gate | **Trivial policy gate**; just returns success. No actual enforcement. |

**Consequence**: Policy validation is not consistently applied:
- CLI validates principal IDs inline (could bypass)
- MCP has no real policy gate (policy_gate.py is a no-op)
- Authority evaluator in core/authority.py called inconsistently

**Recommendation**: CONSOLIDATE to single policy validation layer; make MCP policy_gate.py actually enforce policy.

---

### 5. INSTRUCTION FILE CONSTRAINTS (Embedded but Dispersed)

**Severity**: MODERATE | **Impact**: MODERATE | **Complexity**: Low

#### Issues

| File | Constraint | Specification |
|------|-----------|---------------|
| [caracal/packages/caracal-server/caracal/runtime/instructions.md](caracal/packages/caracal-server/caracal/runtime/instructions.md#L14-L20) | Hard-cut preflight execution | "Preflight must run before any route handler accepts requests" — but checked in three places (runtime/entrypoints, server/main, gateway/proxy) |
| [caracal/packages/caracal-server/caracal/runtime/instructions.md](caracal/packages/caracal-server/caracal/runtime/instructions.md#L20-L24) | Authorization | "All route inputs must be validated… Authentication verified on every protected endpoint" — but done inline in handlers, not at boundary |

**Recommendation**: Embed constraints at framework level (FastAPI middleware) rather than per-route.

---

### 6. BYPASSABLE CHECKS: Silent Fallback Paths

**Severity**: CRITICAL | **Impact**: HIGH | **Complexity**: Moderate

#### Issues

| Function | Bypassable Path | Impact |
|----------|-----------------|--------|
| [caracal/packages/caracal/caracal/runtime/entrypoints.py](caracal/packages/caracal/caracal/runtime/entrypoints.py#L2125) | `_caller_has_capability()` — fallback to DB if JWT is missing/invalid | Cap check could succeed via DB lookup even if token is bad |
| [caracal/packages/caracal/caracal/runtime/entrypoints.py](caracal/packages/caracal/caracal/runtime/entrypoints.py#L2195-L2210) | `_require_active_principal()` — silently returns normalized ID if identity service unavailable | Principal status validation skipped on service error |
| [caracal/packages/caracal-server/caracal/mcp/adapter.py](caracal/packages/caracal-server/caracal/mcp/adapter.py#L1750) | `_authorize_principal_request()` — returns boolean instead of raising; calling code may not check result | Authorization failure could be ignored |

**Consequence**: Authorization checks can be silently bypassed if dependencies fail or if result is not checked.

**Recommendation**: REWRITE to explicit raise-on-fail exceptions; no silent fallbacks.

---

### 7. FUNCTION BOUNDARY CONSTRAINT EMBEDDING GAPS

**Severity**: MODERATE | **Impact**: MODERATE | **Complexity**: Moderate

#### Issues

| Boundary Layer | Gap | Impact |
|---|---|---|
| **Data entry (request handlers)** | Authorization checks embedded in handler code, not in dedicated validation layer | Constraints applied inconsistently; easy to miss |
| **Service layer** | Authorization checks scattered in core/authority.py and mcp/adapter.py; no unified dispatcher | Policy evaluation bypassed if wrong layer called |
| **Data exit (response)** | No constraint verification before returning sensitive data (no audit of what was exposed) | No central place to enforce output constraints |

**Recommendation**: Implement three-layer constraint enforcement:
1. **Entry**: Request validator (middleware/decorator)
2. **Business**: Policy evaluator (core/authority.py)
3. **Exit**: Response filter (response middleware)

---

## Consolidated Finding Summary

| Category | Count | Severity | Recommendation |
|----------|-------|----------|---|
| **Rule sprawl (hardcoded constants)** | 40+ | MODERATE | Centralize to registry |
| **Scattered authorization checks** | 5 locations | CRITICAL | Consolidate to single evaluator |
| **Duplicated validation logic** | 3-4 patterns | MODERATE | Extract to utility layer |
| **Bypassable checks** | 3 functions | CRITICAL | Raise on failure; no fallbacks |
| **Unverified instruction constraints** | 2 instruction files | MODERATE | Embed in framework (middleware) |
| **Incomplete hardcut contracts** | 2 separate systems (OSS/Enterprise) | HIGH | Create unified contract checker |

---

## Recommended Fixes

### Priority 1: CRITICAL (Implement Immediately)

#### 1A. Consolidate Authorization to Single Boundary [REWRITE]
- **File**: Rewrite [caracal/packages/caracal-server/caracal/core/authority.py](caracal/packages/caracal-server/caracal/core/authority.py) to be the ONLY authorization decision point
- **Action**: Move `_authorize_caller_for_target()` logic from [entrypoints.py](caracal/packages/caracal/caracal/runtime/entrypoints.py#L2157) to core/authority.py as public method
- **Action**: Remove inline authorization from [mcp/service.py](caracal/packages/caracal-server/caracal/mcp/service.py#L273-L445) and [mcp/adapter.py](caracal/packages/caracal-server/caracal/mcp/adapter.py#L1725); call core/authority evaluator instead
- **Severity**: CRITICAL
- **Files affected**: 3 (entrypoints, service, adapter)
- **Testing**: Add tests verifying authorization ONLY through core/authority.py

#### 1B. Eliminate Dual-Path Capability Checking [REWRITE]
- **Function**: `_caller_has_capability()` in [entrypoints.py](caracal/packages/caracal/caracal/runtime/entrypoints.py#L2125)
- **Action**: Remove database fallback path; enforce capability from JWT ONLY or raise `SessionValidationError`
- **Consequence**: Fail-closed: if JWT missing/invalid, deny immediately; do not retry via DB
- **Severity**: CRITICAL
- **Testing**: Unit test verifying no DB fallback exists

#### 1C. Remove Silent Fallbacks [PATCH]
- **Locations**: Lines 2195-2210 in [entrypoints.py](caracal/packages/caracal/caracal/runtime/entrypoints.py)
- **Action**: Change `_require_active_principal()` from silent-return to raise on validation failure
- **Current Code**: `if identity_service.get_principal(…) is None: raise …` (good); but also `lifecycle_status` check has return path (bad)
- **Severity**: CRITICAL
- **Testing**: All error conditions must raise; no return paths for failures

---

### Priority 2: HIGH (Implement Within Sprint)

#### 2A. Unify Hardcut Contracts into Single Validator [REWRITE]
- **Files**: 
  - [caracal/packages/caracal/caracal/runtime/hardcut_preflight.py](caracal/packages/caracal/caracal/runtime/hardcut_preflight.py) (OSS)
  - [caracalEnterprise/services/api/src/caracal_api/main.py](caracalEnterprise/services/api/src/caracal_api/main.py#L70) (Enterprise)
- **New function**: `assert_complete_hardcut_contract()` that calls:
  1. `assert_runtime_hardcut()` 
  2. `assert_enterprise_hardcut()`
  3. `_assert_registration_state_hardcut_contract()` (from enterprise main.py)
- **Severity**: HIGH
- **Decision**: **REWRITE** — consolidate into single entry point with clear sequencing

#### 2B. Create Rule Registry (Replace Hardcoded Constants) [REWRITE]
- **File**: Create new [caracal/packages/caracal/caracal/runtime/hardcut_rules.py](caracal/packages/caracal/caracal/runtime/hardcut_rules.py)
- **Action**: Extract 40+ hardcoded marker tuples into `HardcutRuleSet` dataclass:
  ```python
  @dataclass(frozen=True)
  class HardcutRuleSet:
      version: str = "1.0"
      forbidden_sqlite_prefixes: tuple[str, ...] = ("sqlite://", "sqlite+")
      forbidden_sync_markers: tuple[str, ...] = (…)
      forbidden_env_vars: tuple[str, ...] = (…)
      required_vault_env: tuple[str, ...] = (…)
      # ... 40+ rules
  ```
- **Action**: Add versioning; allow rule overrides for testing
- **Severity**: HIGH
- **Decision**: **REWRITE** — improves maintainability and testability

#### 2C. Centralize Policy Validation Layer [REWRITE]
- **New file**: [caracal/packages/caracal-server/caracal/core/policy_validator.py](caracal/packages/caracal-server/caracal/core/policy_validator.py)
- **Move to**: 
  - `_normalize_principal_id()` 
  - `validate_provider_scopes()`
  - All schema validation from authority_policy.py CLI
- **Call from**: CLI, flow, API routes consistently
- **Severity**: HIGH
- **Decision**: **REWRITE** — extract scattered validation into single module

#### 2D. Add FastAPI Middleware for Request/Response Constraints [PATCH]
- **New file**: [caracal/packages/caracal-server/caracal/mcp/constraint_middleware.py](caracal/packages/caracal-server/caracal/mcp/constraint_middleware.py)
- **Implement**:
  - Entry middleware: Validate all route inputs against schema
  - Authorization middleware: Call core/authority.py for ALL protected routes
  - Exit middleware: Filter response to ensure no policy leak
- **Attach to**: All FastAPI routes in service.py
- **Severity**: HIGH
- **Decision**: **PATCH** — decorato model; wrap existing routes

---

### Priority 3: MODERATE (Schedule for Next Sprint)

#### 3A. Embed Instruction Constraints in Code [PATCH]
- **Current**: Constraints documented in .instructions.md files but not enforced
- **Action**: Create `@enforce_hardcut_preflight` decorator for route handlers requiring preflight
- **Action**: Create `@require_authorization` decorator for protected routes
- **Severity**: MODERATE
- **Decision**: **PATCH** — add decorator enforcement

#### 3B. Eliminate policy_gate.py No-Op [PATCH]
- **File**: [caracal/packages/caracal-server/caracal/mcp/policy_gate.py](caracal/packages/caracal-server/caracal/mcp/policy_gate.py)
- **Current**: Returns `{"result": "authorized", ...}` without checking anything
- **Action**: Implement real policy enforcement or remove and consolidate into adapter.py
- **Severity**: MODERATE
- **Decision**: **REWRITE** — make policy_gate actually enforce policy using core/authority.py

#### 3C. Audit Logging for Authorization Decisions [PATCH]
- **Location**: core/authority.py and middleware
- **Add**: Structured logging of all authorization decisions (allow/deny, reason code, principal, resource)
- **Severity**: MODERATE
- **Decision**: **PATCH** — add telemetry

---

## Impact Analysis

| Fix | Files Changed | Complexity | Test Coverage | Risk |
|-----|---|---|---|---|
| 1A: Consolidate Authorization | 3-4 files | HIGH | MUST add 20+ unit tests | MEDIUM (well-scoped) |
| 1B: Remove Dual-Path Checks | 1 file | MEDIUM | 10+ tests | HIGH (authorization logic) |
| 1C: Remove Silent Fallbacks | 1 file | MEDIUM | 5+ tests | MEDIUM |
| 2A: Unify Hardcut Contracts | 2 files | MEDIUM | 10+ tests | LOW (additive) |
| 2B: Create Rule Registry | New file + changes to hardcut_preflight.py | HIGH | 20+ tests | MEDIUM |
| 2C: Centralize Policy Validation | New file + CLI/flow updates | HIGH | 30+ tests | HIGH |
| 2D: Add Middleware | New file + service.py | HIGH | 40+ tests | HIGH |
| 3A: Instruction Decorators | New decorators | MEDIUM | 15+ tests | LOW |
| 3B: Real Policy Gate | 1 file rewrite | MEDIUM | 10+ tests | MEDIUM |
| 3C: Audit Logging | 2-3 files | LOW | 5+ tests | LOW |

---

## Consolidation Decision Matrix

| Area | Current State | Consolidation Strategy | Rewrite vs Patch |
|------|---|---|---|
| **Hardcut preflight** | 3 separate assert functions; enterprise adds 4th | Create `assert_complete_hardcut_contract()` with versioning | **REWRITE** |
| **Authorization enforcement** | 5 locations with different logic | Single core/authority.py evaluator; middleware for HTTP layer | **REWRITE** + **PATCH** |
| **Policy validation** | Scattered CLI, flow, API paths | Central policy_validator.py module; consistent entry point | **REWRITE** |
| **Constraint embedding** | Documented in .md, not enforced | FastAPI middleware + decorators | **PATCH** |
| **Rule definitions** | 40+ hardcoded module constants | HardcutRuleSet registry with versioning | **REWRITE** |

---

## Conclusion

**Current Architecture Risk**: MODERATE-HIGH
- Hardcut/preflight validation is centralized but lacks versioning and incomplete for enterprise
- Authorization logic bypassed in 5+ locations with dual-path fallbacks
- Policy validation not consistently applied; MCP policy gate is a no-op
- Constraints documented but not embedded at framework boundaries

**Post-Consolidation Risk**: LOW
- Single authorization evaluator with fail-closed semantics
- Middleware enforces constraints at entry/exit
- Unified hardcut contract with versioning
- Policy validation layer consistent across CLI/flow/API

**Estimated Effort**: 3-4 sprints (prioritize Tier-1 critical fixes first)

---

**Report Generated**: 2026-04-29 | **Scope**: OSS + Enterprise | **Audit Type**: Read-only Hardcut/Preflight/Policy Sprawl Scan