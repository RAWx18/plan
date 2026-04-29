Based on my audit of the security policy enforcement across both repositories, here is the comprehensive report:

---

## SECURITY POLICY INTEGRATION AUDIT REPORT

### EXECUTIVE SUMMARY

The codebase has centralized hard-cut preflight enforcement in [caracal/runtime/hardcut_preflight.py](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py), but contains significant **rule sprawl**, **bypassable constraints**, and **string-based checks that should be type constraints**. Enterprise feature gates are properly isolated, but several OSS policy checks can be bypassed via flags or are inconsistently applied.

---

## 1. RULE SPRAWL — External Lists That Should Be Core Constraints

### 1.1 Secret Backend Allow-List (EMBED_IN_CORE)
**Location**: [hardcut_preflight.py:83](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L83)
```python
_ALLOWED_SECRET_BACKENDS = ("vault",)
```
**Issue**: Secret backend validation is a string check in a separate file.  
**Where it SHOULD live**: `caracal.core.secret_backend` module as an enum with type enforcement.  
**Bypassable**: No (hard fails), but check only runs during startup preflight.  
**Type**: RULE_SPRAWL  
**Recommendation**: MAKE_TYPE_CONSTRAINT — Create `SecretBackendType(Enum)` and enforce at usage site in provider/credential modules.

---

### 1.2 Forbidden Env Var List (EMBED_IN_CORE)
**Location**: [hardcut_preflight.py:74-81](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L74-L81)
```python
_FORBIDDEN_COMPAT_ENV_VARS = (
    "CCL_COMPAT_ON",
    "CCL_COMPAT_ALIASES",
    ...
)
```
**Issue**: Compatibility mode flags are only checked at startup. If set mid-execution or via import, they're not blocked.  
**Where it SHOULD live**: Each compatibility feature should fail at usage site with explicit error, not via env var scan.  
**Bypassable**: Yes — set after startup, or used in modules that don't call preflight.  
**Type**: WRONG_LOCATION  
**Recommendation**: EMBED_IN_CORE — Remove env var checks; make compatibility features raise errors directly when invoked.

---

### 1.3 SQLite Forbidden Prefixes (MAKE_TYPE_CONSTRAINT)
**Location**: [hardcut_preflight.py:27](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L27), [hardcut_preflight.py:99-100](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L99-L100)
```python
_FORBIDDEN_SQLITE_PREFIXES = ("sqlite://", "sqlite+")
```
**Issue**: Database engine is validated via string prefix matching, not type system.  
**Where it SHOULD live**: Database connection factory — reject SQLite at connection creation, not in preflight scanner.  
**Bypassable**: No (hard fails at startup).  
**Type**: RULE_SPRAWL  
**Recommendation**: MAKE_TYPE_CONSTRAINT — Use SQLAlchemy dialect checks in `db.connection` module.

---

### 1.4 Forbidden Compose File Markers (WRONG_LOCATION)
**Location**: [hardcut_preflight.py:28-73](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L28-L73)
```python
_FORBIDDEN_RUNTIME_COMPOSE_MARKERS = (
    "caracal_state:",
    "/home/caracal/.caracal",
)
_REQUIRED_RUNTIME_COMPOSE_MARKERS = (
    "  vault:",
    "image: ${ccl_vault_sidecar_image",
    ...
)
```
**Issue**: Docker Compose contract enforcement via text scanning. Fragile (whitespace-sensitive), bypassed if compose file name changes.  
**Where it SHOULD live**: Startup should parse compose YAML and validate structure, not grep for strings.  
**Bypassable**: Yes — rename compose file or disable preflight.  
**Type**: WRONG_LOCATION  
**Recommendation**: CENTRALIZE — Use `docker-compose config` output or YAML parser to validate structure.

---

### 1.5 Session Signing Algorithm Allow-List (MAKE_TYPE_CONSTRAINT)
**Location**: [hardcut_preflight.py:92](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L92)
```python
_ALLOWED_SESSION_SIGNING_ALGORITHMS = ("RS256", "ES256")
```
**Issue**: JWT algorithm validation is a string check, not enforced in JWT issuer.  
**Where it SHOULD live**: JWT token signing module should have enum-based algorithm selection.  
**Bypassable**: Partially — preflight catches it, but JWT module doesn't enforce at issuance.  
**Type**: RULE_SPRAWL  
**Recommendation**: MAKE_TYPE_CONSTRAINT — Create `SessionSigningAlgorithm(Enum)` and enforce in JWT signing path.

---

### 1.6 Forbidden Config Markers (REDUNDANT)
**Location**: [hardcut_preflight.py:93-112](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L93-L112)
```python
_FORBIDDEN_CONFIG_MARKERS = (
    "enterprise.json",
    "workspaces.json",
    "sqlite://",
    "sqlite+",
    "caracal_principal_key_backend=local",
    ...
)
```
**Issue**: Overlaps with other checks (SQLite, secret backends). String matching in config files.  
**Where it SHOULD live**: Each config option should fail at parse/load time, not via text scan.  
**Bypassable**: Yes — malformed configs may not be scanned.  
**Type**: REDUNDANT  
**Recommendation**: REMOVE — Already covered by other constraints; config parser should validate.

---

## 2. REDUNDANT POLICY DEFINITIONS

### 2.1 SQLite Prohibition (3 places)
**Locations**:
1. [hardcut_preflight.py:27](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L27) — `_FORBIDDEN_SQLITE_PREFIXES`
2. [hardcut_preflight.py:99-100](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L99-L100) — `_FORBIDDEN_CONFIG_MARKERS` contains `sqlite://`, `sqlite+`
3. [settings.py:1108](Caracal/packages/caracal-server/caracal/config/settings.py#L1108) — Merkle signing backend check enforces PostgreSQL

**Consolidation**: EMBED_IN_CORE — Single database engine type constraint at connection factory.

---

### 2.2 Vault Backend Enforcement (OSS + Enterprise)
**Locations**:
1. [hardcut_preflight.py:249-278](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L249-L278) — `_secret_backend_violations`
2. [settings.py:1096-1122](Caracal/packages/caracal-server/caracal/config/settings.py#L1096-L1122) — Merkle config validation enforces vault
3. [Enterprise config.py:59](caracalEnterprise/services/api/src/caracal_api/config.py#L59) — Default backend set to `vault`

**Consolidation**: CENTRALIZE — Single vault backend enforcement in shared `caracal.vault` module.

---

### 2.3 Compatibility Env Var Prohibition (2 places)
**Locations**:
1. [hardcut_preflight.py:74-81](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L74-L81) — List of forbidden compat vars
2. [hardcut_forbidden_marker_scan.py:70-73](Caracal/scripts/hardcut_forbidden_marker_scan.py#L70-L73) — Scans for same patterns in files

**Consolidation**: REMOVE — Static scanner is sufficient; runtime check is redundant.

---

## 3. BYPASSABLE RULES

### 3.1 JSONB Check Flag (CRITICAL BYPASS)
**Location**: [hardcut_preflight.py:438](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L438), [hardcut_preflight.py:470](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L470), [hardcut_preflight.py:502](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L502)
```python
def assert_runtime_hardcut(..., check_jsonb: bool = True):
def assert_enterprise_hardcut(..., check_jsonb: bool = True):
def assert_migration_hardcut(..., check_jsonb: bool = True):
```
**Bypass Pattern**: [migration.py:50](Caracal/packages/caracal-server/caracal/cli/migration.py#L50) — `check_jsonb=False` disables JSONB enforcement for migration CLI.  
**Issue**: Migration commands can bypass relational-only enforcement. Tests also bypass it pervasively.  
**Bypassable**: **YES** — Pass `check_jsonb=False` flag.  
**Type**: BYPASSABLE  
**Recommendation**: REMOVE — JSONB usage should fail at ORM model definition time, not via preflight flag.

---

### 3.2 Development Mode Bypass (CRITICAL BYPASS)
**Location**: [Enterprise config.py:148](caracalEnterprise/services/api/src/caracal_api/config.py#L148), [middleware/license.py:72](caracalEnterprise/services/api/src/caracal_api/middleware/license.py#L72)
```python
development_mode: bool = False
...
if settings.development_mode:
    return await call_next(request)  # Skip license check
```
**Bypass Pattern**: Set `DEVELOPMENT_MODE=true` or `development_mode: true` in config.  
**Issue**: License enforcement is skipped entirely in dev mode.  
**Bypassable**: **YES** — Environment variable.  
**Type**: BYPASSABLE (intentional but undocumented).  
**Recommendation**: CENTRALIZE — Document this as explicit policy; add audit logging when bypassed.

---

### 3.3 Local Vault Mode Prohibition (INCONSISTENT)
**Location**: [hardcut_preflight.py:273-278](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L273-L278)
```python
vault_mode = (env_vars.get(_VAULT_MODE_ENV) or "").strip().lower()
if vault_mode in _LOCAL_VAULT_MODE_VALUES:
    violations.append(...)
```
**Issue**: Only blocks `CCL_VAULT_MODE=local` in **runtime** paths. Not enforced in Enterprise API startup.  
**Bypassable**: **YES** — Set after startup or in non-runtime paths.  
**Type**: INCONSISTENT  
**Recommendation**: EMBED_IN_CORE — Vault client should reject local mode at connection time.

---

### 3.4 Gateway Execution Exclusivity (PARTIAL ENFORCEMENT)
**Location**: [hardcut_preflight.py:285-307](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L285-L307)
```python
def _execution_exclusivity_violations(env_vars):
    # Checks CCL_GATEWAY_ENABLED vs CCL_GATEWAY_URL conflicts
```
**Issue**: Only validates env vars at startup. Doesn't prevent runtime switching or module-level overrides.  
**Bypassable**: **YES** — Change execution mode after startup.  
**Type**: INCONSISTENT  
**Recommendation**: EMBED_IN_CORE — Execution mode should be singleton, set once at app init.

---

## 4. STRING CHECKS THAT SHOULD BE TYPE CONSTRAINTS

| **Check** | **Location** | **Current Implementation** | **Should Be** |
|-----------|--------------|---------------------------|---------------|
| Secret backend | [hardcut_preflight.py:83](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L83) | `_ALLOWED_SECRET_BACKENDS = ("vault",)` | `SecretBackend(Enum)` in `caracal.vault` |
| JWT algorithm | [hardcut_preflight.py:92](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L92) | `_ALLOWED_SESSION_SIGNING_ALGORITHMS = ("RS256", "ES256")` | `SessionAlgorithm(Enum)` in `caracal.auth` |
| Database engine | [hardcut_preflight.py:27](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L27) | `_FORBIDDEN_SQLITE_PREFIXES = ("sqlite://", "sqlite+")` | SQLAlchemy engine type validation |
| Vault mode | [hardcut_preflight.py:94](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L94) | `_LOCAL_VAULT_MODE_VALUES = ("local", "dev", "development")` | `VaultMode(Enum)` in vault client |
| Tier names | [Enterprise pricing_config.py:24](caracalEnterprise/services/api/src/caracal_api/pricing_config.py#L24) | **Already enum** — `Tier(str, Enum)` ✅ | N/A (correct) |
| Feature flags | [Enterprise pricing_config.py:120](caracalEnterprise/services/api/src/caracal_api/pricing_config.py#L120) | `FeatureFlags` dataclass with string keys | Enum-based feature registry |

---

## 5. INCONSISTENT POLICY APPLICATION

### 5.1 Tool Registration Policy (INCONSISTENT)
**Check Location**: [tool_registry_contract.py:89-205](Caracal/packages/caracal-server/caracal/mcp/tool_registry_contract.py#L89-L205) — `validate_active_tool_mappings`  
**Issue**: Tool validation only runs on explicit CLI invocation or deployment checks. Not enforced at tool registration time.  
**Where it's skipped**: MCP tool registration endpoint accepts invalid tools; validation is deferred.  
**Type**: INCONSISTENT  
**Recommendation**: EMBED_IN_CORE — Validate tools at registration time in [mcp/service.py:867](Caracal/packages/caracal-server/caracal/mcp/service.py#L867).

---

### 5.2 Provider Scope Validation (INCONSISTENT)
**Check Location**: [provider_scopes.py:124-156](Caracal/packages/caracal-server/caracal/cli/provider_scopes.py#L124-L156) — `validate_provider_scopes`  
**Issue**: Validation only runs in CLI commands (authority policy create, mandate issue). Not enforced in:
- Direct DB inserts
- API endpoints (Enterprise)
- TUI flows (unless explicitly called)

**Type**: INCONSISTENT  
**Recommendation**: EMBED_IN_CORE — Move validation to `AuthorityPolicy` model `__init__` or ORM validator.

---

### 5.3 Merkle Signing Backend (FULLY ENFORCED ✅)
**Check Location**: [settings.py:1096-1122](Caracal/packages/caracal-server/caracal/config/settings.py#L1096-L1122)  
```python
if config.merkle.signing_backend != "vault":
    raise InvalidConfigurationError(...)
```
**Status**: ✅ Properly enforced at config load time. Cannot be bypassed.

---

### 5.4 Feature Gate Enforcement (PROPERLY ISOLATED ✅)
**Check Location**: [Enterprise feature_gate.py:85-246](caracalEnterprise/services/api/src/caracal_api/services/feature_gate.py#L85-L246)  
**Enforcement Points**:
- Route handlers call `FeatureGate.require_feature()` / `enforce_limit()`
- Middleware meters API calls: [tier_enforcement.py:30-143](caracalEnterprise/services/api/src/caracal_api/middleware/tier_enforcement.py#L30-L143)
- Checks subscription status: [feature_gate.py:93-101](caracalEnterprise/services/api/src/caracal_api/services/feature_gate.py#L93-L101)

**Status**: ✅ Properly layered. No bypass paths found.

---

## 6. POLICY REGISTRATION — WHO CAN MODIFY?

### 6.1 Authority Policy Creation
**CLI Path**: [authority_policy.py:87-220](Caracal/packages/caracal-server/caracal/cli/authority_policy.py#L87-L220) — `caracal policy create`  
**TUI Path**: [authority_policy_flow.py:138-202](Caracal/packages/caracal-server/caracal/flow/screens/authority_policy_flow.py#L138-L202)  
**Enterprise API Path**: [policies.py:241](caracalEnterprise/services/api/src/caracal_api/routes/policies.py#L241) — `POST /api/policies`  
**Constraints**:
- Principal must exist in DB
- Scopes validated against workspace provider catalog
- `created_by` field set to `"cli"` or API user ID

**Who can modify**: Any authenticated CLI/TUI/API user with DB access. **No RBAC enforcement** — policy creation is not gated by authority checks.

---

### 6.2 Tool Registration
**Registration Endpoint**: [mcp/service.py:867-920](Caracal/packages/caracal-server/caracal/mcp/service.py#L867-L920) — `register_tool`  
**Required Capability**: `mcp.tool_registry.manage` — enforced via policy gate at [policy_gate.py:9-12](Caracal/packages/caracal-server/caracal/mcp/policy_gate.py#L9-L12).  
**Issue**: Policy gate is a **passthrough** — returns success without actual enforcement:
```python
def passthrough(**kwargs: object) -> dict[str, object]:
    return {"result": "authorized", "args": kwargs}
```
**Who can modify**: Anyone with MCP adapter access. **Policy check is a no-op.**

---

## 7. DUPLICATION MAP — OSS vs ENTERPRISE

| **Check** | **OSS Location** | **Enterprise Location** | **Duplicated?** |
|-----------|-----------------|------------------------|----------------|
| Hard-cut preflight | [hardcut_preflight.py](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py) | [main.py:283](caracalEnterprise/services/api/src/caracal_api/main.py#L283) calls OSS | ❌ Shared |
| Vault enforcement | [hardcut_preflight.py:249](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L249) | [config.py:59](caracalEnterprise/services/api/src/caracal_api/config.py#L59) default | ✅ Redundant |
| Feature gates | N/A | [feature_gate.py](caracalEnterprise/services/api/src/caracal_api/services/feature_gate.py) | ❌ Enterprise-only |
| Tier enforcement | N/A | [tier_enforcement.py](caracalEnterprise/services/api/src/caracal_api/middleware/tier_enforcement.py) | ❌ Enterprise-only |
| Provider validation | [provider_scopes.py](Caracal/packages/caracal-server/caracal/cli/provider_scopes.py) | Used by Enterprise API routes | ❌ Shared |
| Tool registry validation | [tool_registry_contract.py](Caracal/packages/caracal-server/caracal/mcp/tool_registry_contract.py) | Called from OSS MCP service | ❌ Shared |

---

## 8. FLAGS/ENV VARS THAT DISABLE CHECKS

| **Flag/Env Var** | **What It Bypasses** | **Intentional?** | **Location** |
|------------------|---------------------|-----------------|--------------|
| `check_jsonb=False` | JSONB model enforcement | ⚠️ Test/migration only | [hardcut_preflight.py:438](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L438) |
| `development_mode: true` | License checks in Enterprise | ✅ Intentional | [config.py:148](caracalEnterprise/services/api/src/caracal_api/config.py#L148) |
| `CCL_VAULT_MODE=local` | Vault production mode | ⚠️ Dev only, but not always blocked | [hardcut_preflight.py:273](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L273) |
| `CCL_GATEWAY_ENABLED=false` | Gateway execution mode validation | ✅ Intentional | [hardcut_preflight.py:296](Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L296) |

---

## SUMMARY OF RECOMMENDATIONS

### HIGH PRIORITY (Security Impact)
1. **Remove `check_jsonb` bypass** — JSONB should fail at model definition time
2. **Remove passthrough policy gate** — [policy_gate.py](Caracal/packages/caracal-server/caracal/mcp/policy_gate.py) doesn't enforce anything
3. **Add RBAC to policy creation** — Authority policy creation has no permission checks
4. **Enforce tool validation at registration** — Not at deferred CLI validation

### MEDIUM PRIORITY (Architecture)
5. **Convert secret backend to enum** — Replace string checks with `SecretBackend(Enum)`
6. **Convert JWT algorithm to enum** — Replace string checks with `SessionAlgorithm(Enum)`
7. **Move SQLite check to connection factory** — Not preflight scanner
8. **Consolidate vault enforcement** — Single source of truth in `caracal.vault`

### LOW PRIORITY (Cleanup)
9. **Remove redundant config marker checks** — Already covered by other validators
10. **Replace compose file text scanning** — Use YAML parser validation
11. **Document `development_mode` bypass** — Audit log when license checks skipped