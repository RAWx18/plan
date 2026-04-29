Based on my comprehensive scan of both repositories, here is the **Legacy & Dead Code Analysis Report**:

---

## CRITICAL FINDINGS — High Copy-Paste Risk

### 1. LEGACY_COMPAT: Legacy Environment Variable Rewriting System
**Files:**
- [caracal-server/caracal/config/settings.py](file:///home/raw/Documents/workspace/caracalEcosystem/Caracal/packages/caracal-server/caracal/config/settings.py#L107-L170)

**Type:** LEGACY_COMPAT + FALLBACK

**Description:**
Complete legacy env placeholder rewriting system (`_LEGACY_ENV_PLACEHOLDER_RENAMES`, `_rewrite_legacy_env_placeholders`, `_normalize_legacy_config_data`) that converts old `CARACAL_*` env vars to new `CCL_*` names. This is a silent compatibility layer that rewrites config files on load.

**Symbols:**
- `_LEGACY_ENV_PLACEHOLDER_RENAMES` (dict, lines 107-120)
- `_rewrite_legacy_env_placeholders()` (function, lines 123-153)
- `_normalize_legacy_config_data()` (function, lines 156-170)

**Copy-Paste Trap Risk:** HIGH — AI may suggest using old `CARACAL_*` env vars from comments or seeing this mapping

**Removal Action:** SAFE_DELETE — Hard-cut mode forbids legacy env vars (enforced in runtime/hardcut_preflight.py)

---

### 2. LEGACY_COMPAT: Workspace Config Repair Function
**File:** [caracal-server/caracal/config/settings.py](file:///home/raw/Documents/workspace/caracalEcosystem/Caracal/packages/caracal-server/caracal/config/settings.py#L851-L915)

**Type:** FALLBACK + AUTO_REPAIR

**Description:**
`_attempt_legacy_workspace_config_repair()` attempts to auto-repair malformed YAML files from old onboarding flows. Called silently on YAML parse failures.

**Symbol:** `_attempt_legacy_workspace_config_repair(config_path: str) -> bool`

**Copy-Paste Trap Risk:** MEDIUM — Suggests that malformed configs are acceptable

**Removal Action:** REWRITE_CALLER — Remove auto-repair; fail loudly on malformed YAML. Onboarding now generates valid YAML (lines 700-740 in flow/screens/onboarding.py confirm this).

---

### 3. DEAD_CONFIG: CompatibilityConfig Class
**File:** [caracal-server/caracal/config/settings.py](file:///home/raw/Documents/workspace/caracalEcosystem/Caracal/packages/caracal-server/caracal/config/settings.py#L350-L368)

**Type:** OBSOLETE_CONFIG + ALWAYS_TRUE

**Description:**
```python
class CompatibilityConfig:
    """Redis caching and Merkle integrity are mandatory in current Caracal
    deployments. These fields are retained only for backward compatibility
    with older config files."""
    enable_merkle: bool = True
    enable_redis: bool = True
```

Both fields are **always True** and never read with false branches. The docstring admits they're kept "only for backward compatibility."

**Copy-Paste Trap Risk:** HIGH — AI may generate code that checks `if config.compatibility.enable_merkle` when it's always true

**Removal Action:** SAFE_DELETE — Remove class, update CaracalConfig to drop the field

---

### 4. FALLBACK: Runtime Password Recovery Function
**File:** [caracal-server/caracal/flow/screens/onboarding.py](file:///home/raw/Documents/workspace/caracalEcosystem/Caracal/packages/caracal-server/caracal/flow/screens/onboarding.py#L1243-L1250)

**Type:** FALLBACK + RECOVERY_PATH

**Symbol:** `_try_runtime_password_fallback()`

**Description:**
Fallback that attempts to recover DB password from runtime environment when workspace config is corrupted. Called at line 1787 during database onboarding.

**Copy-Paste Trap Risk:** MEDIUM — Suggests dual password sources are valid

**Removal Action:** MERGE_INTO — Consolidate password resolution into single source (env → .env → config.yaml), no fallback recovery

---

### 5. DUPLICATE: Multiple PostgreSQL Start Methods
**File:** [caracal-server/caracal/flow/screens/onboarding.py](file:///home/raw/Documents/workspace/caracalEcosystem/Caracal/packages/caracal-server/caracal/flow/screens/onboarding.py#L1150-L1235)

**Type:** DUPLICATE_LOGIC

**Symbol:** `_start_postgresql()` with method="auto|docker|systemctl|pg_ctl"

**Description:**
Three separate PostgreSQL start implementations with redundant error handling. Onboarding documentation shows only docker-compose is officially supported.

**Copy-Paste Trap Risk:** MEDIUM — Suggests systemctl/pg_ctl are valid runtime paths when they're dev-only

**Removal Action:** REWRITE — Keep docker-compose path only, remove systemctl/pg_ctl branches

---

### 6. DEAD_FUNC: CLI Fallback Display
**File:** [caracal-server/caracal/flow/app.py](file:///home/raw/Documents/workspace/caracalEcosystem/Caracal/packages/caracal-server/caracal/flow/app.py#L531-L540)

**Type:** DEAD_FUNC + UNIMPLEMENTED_STUB

**Symbol:** `_show_cli_fallback(self, group: str, command: str) -> None`

**Description:**
Displays "this feature is not implemented in Flow, use CLI" messages. Called from lines 220, 249, 274 but all branches are unreachable (action handlers exist).

**Copy-Paste Trap Risk:** LOW

**Removal Action:** SAFE_DELETE — Remove function and all call sites after confirming no unimplemented features remain

---

### 7. FALLBACK: Multiple .env File Search Locations
**File:** [caracal-server/caracal/flow/screens/onboarding.py](file:///home/raw/Documents/workspace/caracalEcosystem/Caracal/packages/caracal-server/caracal/flow/screens/onboarding.py#L45-L90)

**Type:** FALLBACK + SEARCH_CHAIN

**Symbol:** `_find_env_file() -> Optional[Path]`

**Description:**
Searches 3+ locations for .env file: CWD, project root (walk 10 levels up), CWD ancestors (5 levels). Container runtime only needs host-shared path.

**Copy-Paste Trap Risk:** MEDIUM — Multiple search paths suggest ambiguous .env location is acceptable

**Removal Action:** REWRITE — Single canonical location (workspace root or CCL_HOME), fail loudly if missing

---

### 8. DUPLICATE: Enterprise URL Resolution
**File:** [caracal-server/caracal/deployment/enterprise_runtime.py](file:///home/raw/Documents/workspace/caracalEcosystem/Caracal/packages/caracal-server/caracal/deployment/enterprise_runtime.py#L53-L104)

**Type:** DUPLICATE_LOGIC + FALLBACK_CHAIN

**Functions:**
- `_load_workspace_dotenv()` (lines 53-90)
- `_read_env()` (lines 93-98)
- `_resolve_api_url()` (lines 240-260)

**Description:**
Three overlapping functions that resolve env vars from process env, workspace .env, and enterprise config with different precedence orders.

**Copy-Paste Trap Risk:** HIGH — Inconsistent precedence across resolution functions

**Removal Action:** MERGE_INTO — Single env resolution function with explicit precedence

---

## MEDIUM PRIORITY — Legacy Patterns

### 9. LEGACY_COMMENT: "Legacy .db files left in place"
**File:** [caracal-server/caracal/flow/screens/onboarding.py](file:///home/raw/Documents/workspace/caracalEcosystem/Caracal/packages/caracal-server/caracal/flow/screens/onboarding.py#L736)

**Type:** COMMENTED_LEGACY

**Description:**
```python
# Note: SQLite no longer supported — PostgreSQL only.
# Legacy .db files are left in place (harmless) for manual cleanup.
```

**Copy-Paste Trap Risk:** MEDIUM — Suggests SQLite files might exist and be valid

**Removal Action:** SAFE_DELETE — Remove comment; SQLite is forbidden by hardcut_preflight.py

---

### 10. FORBIDDEN_PATTERNS: Hard-Cut Preflight Blocklist
**File:** [caracal/runtime/hardcut_preflight.py](file:///home/raw/Documents/workspace/caracalEcosystem/Caracal/packages/caracal/caracal/runtime/hardcut_preflight.py#L74-L116)

**Type:** OBSOLETE_CHECK + BLOCKLIST

**Description:**
Multiple forbidden pattern lists that check for legacy features:
- `_FORBIDDEN_COMPAT_ENV_VARS` (CCL_COMPAT_ON, CCL_COMPAT_ALIASES, CCL_COMPAT_MODE)
- `_FORBIDDEN_CONFIG_MARKERS` (sqlite://, enterprise.json, workspaces.json)
- Compatibility violation detection (line 230: `_compatibility_violations()`)

**Copy-Paste Trap Risk:** HIGH — Seeing these patterns suggests they were once valid

**Removal Action:** KEEP — These are active guard rails preventing legacy usage. However, simplify error messages to not describe what the forbidden features were.

---

### 11. DUPLICATE_EXCEPTION: Exception Classes with No Usage
**Files:**
- [caracal-server/caracal/exceptions.py](file:///home/raw/Documents/workspace/caracalEcosystem/Caracal/packages/caracal-server/caracal/exceptions.py) (60+ exception classes, all with `pass` body)
- [caracal-server/caracal/deployment/exceptions.py](file:///home/raw/Documents/workspace/caracalEcosystem/Caracal/packages/caracal-server/caracal/deployment/exceptions.py#L1-L80)

**Type:** DEAD_EXCEPTION + EMPTY_IMPL

**Description:**
Exception hierarchy with dozens of specialized exceptions, most with no implementation and potentially unused.

**Symbols (sample):**
- `ConfigurationCorruptedError`
- `ConfigurationNotFoundError`  
- `EventReplayError`
- `SDKConfigurationError`

**Copy-Paste Trap Risk:** MEDIUM — AI may catch unused exceptions

**Removal Action:** AUDIT_USAGE — Use grep to find which exceptions are never raised, then delete unused ones

---

### 12. BARE_EXCEPT: Widespread Error Suppression
**Files:** 20+ files across both repos

**Type:** ERROR_SUPPRESSION

**Pattern:**
```python
except Exception:
    pass  # or: logger.debug(..., exc_info=True)
```

**Locations (sample):**
- [caracal-server/caracal/logging_config.py:188](file:///home/raw/Documents/workspace/caracalEcosystem/Caracal/packages/caracal-server/caracal/logging_config.py#L188)
- [caracal-server/caracal/core/vault.py:79,431,450,474](file:///home/raw/Documents/workspace/caracalEcosystem/Caracal/packages/caracal-server/caracal/core/vault.py#L79)
- [caracal-server/caracal/deployment/config_manager.py:280,297,653,663](file:///home/raw/Documents/workspace/caracalEcosystem/Caracal/packages/caracal-server/caracal/deployment/config_manager.py#L280)
- [caracalEnterprise/services/api/src/caracal_api/main.py:191](file:///home/raw/Documents/workspace/caracalEcosystem/caracalEnterprise/services/api/src/caracal_api/main.py#L191)

**Copy-Paste Trap Risk:** LOW — Linters catch these

**Removal Action:** REWRITE — Replace with specific exception types or fail-closed behavior

---

### 13. CONDITIONAL_IMPORT: Lazy Imports Throughout Codebase
**Pattern:** 20+ instances of `try: import ... except ImportError: pass`

**Files (sample):**
- [caracal-server/caracal/deployment/migration.py:1156](file:///home/raw/Documents/workspace/caracalEcosystem/Caracal/packages/caracal-server/caracal/deployment/migration.py#L1156)
- [caracal-server/caracal/deployment/config_manager.py:273,503,648](file:///home/raw/Documents/workspace/caracalEcosystem/Caracal/packages/caracal-server/caracal/deployment/config_manager.py#L273)
- [caracalEnterprise/services/api/src/caracal_api/routes/compliance.py:94,187](file:///home/raw/Documents/workspace/caracalEcosystem/caracalEnterprise/services/api/src/caracal_api/routes/compliance.py#L94)

**Type:** FALLBACK + CONDITIONAL_FEATURE

**Description:**
Conditional imports suggest optional dependencies when they're mandatory (e.g., `import yaml`, `import tarfile`).

**Copy-Paste Trap Risk:** MEDIUM — Suggests graceful degradation when dependencies should be required

**Removal Action:** REWRITE — Move imports to module level, fail loudly on ImportError

---

### 14. MIGRATION_FUNCTIONS: Edition Migration System
**File:** [caracal-server/caracal/deployment/migration.py](file:///home/raw/Documents/workspace/caracalEcosystem/Caracal/packages/caracal-server/caracal/deployment/migration.py)

**Type:** COMPLEX_MIGRATION + DUAL_EDITION

**Symbols:**
- `migrate_credentials_oss_to_enterprise()` (line 921)
- `migrate_credentials_enterprise_to_oss()` (line 1026)
- `migrate_edition()` (line 236)
- `_migrate_edition_settings()` (line 1174)

**Description:**
Bidirectional migration system between OSS and Enterprise editions. Hard-cut mode forbids edition switching (per hardcut_preflight.py).

**Copy-Paste Trap Risk:** HIGH — Suggests edition switching is a runtime operation when it's a one-time migration

**Removal Action:** DEPRECATE_FEATURE — Document as one-time migration tool, not runtime capability

---

### 15. DEAD_ROUTE: Legacy Sync Routes
**File:** Tests show these are explicitly forbidden

**Evidence:** [caracalEnterprise/services/api/tests/test_hardcut_connection_surface.py](file:///home/raw/Documents/workspace/caracalEcosystem/caracalEnterprise/services/api/tests/test_hardcut_connection_surface.py#L75-L123)

```python
def test_sync_router_has_no_password_auth_fallback_surface() -> None:
    ...
    assert "@router.get(\"/connection-status\")" not in payload
    assert "@router.post(\"/cleanup-license-password\")" not in payload
```

**Type:** DEAD_ROUTE + REMOVED_SURFACE

**Copy-Paste Trap Risk:** MEDIUM — Test asserts routes don't exist, but may see route stubs in git history

**Removal Action:** VERIFIED_REMOVED — Already removed, good

---

### 16. FALLBACK_LOGGING: Duplicate Log Path Resolution
**File:** [caracal-server/caracal/flow/screens/logs_viewer.py](file:///home/raw/Documents/workspace/caracalEcosystem/Caracal/packages/caracal-server/caracal/flow/screens/logs_viewer.py#L77-L190)

**Type:** FALLBACK + SEARCH_CHAIN

**Description:**
```python
# Fallback for plain-text logs.
...
# Legacy/global locations from older layouts.
...
# Resolve log path from active workspace and known fallback locations.
```

Multiple fallback log locations checked in sequence.

**Copy-Paste Trap Risk:** MEDIUM

**Removal Action:** REWRITE — Single log location per workspace, fail if not found

---

### 17. WORKSPACE_IMPORT: Multiple Workspace Validation Chains
**File:** [caracal-server/caracal/flow/screens/onboarding.py](file:///home/raw/Documents/workspace/caracalEcosystem/Caracal/packages/caracal-server/caracal/flow/screens/onboarding.py#L1350-L1500)

**Type:** COMPLEX_VALIDATION + MULTI_PATH

**Description:**
Workspace import action (line 1355+) has multiple fallback paths:
- Import with override name
- Import with original name
- Seed DB config if missing
- Auto-detect imported workspace name from registry diff

**Copy-Paste Trap Risk:** MEDIUM

**Removal Action:** SIMPLIFY — Require explicit workspace name, remove auto-detection

---

## LOW PRIORITY — Minor Issues

### 18. UNUSED_IMPORT: Many try/except import statements never used
- Standard library imports wrapped in try/except (tarfile, yaml, pwd)
- Should be module-level imports

### 19. VALIDATION_DUPLICATE: Multiple validate_* functions with overlapping logic
**Files:** [caracal-server/caracal/core/*.py](file:///home/raw/Documents/workspace/caracalEcosystem/Caracal/packages/caracal-server/caracal/core/)

14+ validation functions found:
- `validate_mandate()` (authority.py:787, mandate.py:477)
- `validate_token()` (delegation.py:210)
- `validate_delegation_direction()` (delegation_graph.py:250)
- `validate_authority_path()` (delegation_graph.py:609)
- Multiple session validation methods (session_manager.py:642,683,700)

**Copy-Paste Trap Risk:** LOW — Different domains

**Removal Action:** DOCUMENT — These are legitimate domain-specific validators

---

## SUMMARY STATISTICS

| Category | Count | Copy-Paste Risk |
|----------|-------|-----------------|
| LEGACY_COMPAT | 3 | HIGH |
| FALLBACK | 6 | HIGH |
| DUPLICATE | 3 | MEDIUM |
| DEAD_FUNC | 2 | LOW |
| ERROR_SUPPRESSION | 20+ | LOW |
| OBSOLETE_CONFIG | 1 | HIGH |
| MIGRATION_SYSTEM | 4 | HIGH |

**Total High-Risk Findings:** 11  
**Total Findings:** 19+ major patterns

---

## RECOMMENDED CLEANUP ORDER

1. **Remove CompatibilityConfig** (settings.py:350) — Always true, misleading
2. **Remove _LEGACY_ENV_PLACEHOLDER_RENAMES system** (settings.py:107-170) — Forbidden by hardcut
3. **Delete _attempt_legacy_workspace_config_repair** (settings.py:851) — Auto-repair hides errors
4. **Simplify PostgreSQL start** (onboarding.py:1150) — Docker-compose only
5. **Consolidate enterprise URL resolution** (enterprise_runtime.py:53-260) — Three overlapping functions
6. **Audit unused exceptions** (exceptions.py) — 60+ classes, many unused
7. **Replace bare except blocks** (20+ files) — Fail closed on unexpected errors
8. **Remove conditional imports** (20+ files) — Dependencies are mandatory
9. **Document migration tools** (migration.py) — Not runtime capabilities