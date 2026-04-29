Based on my thorough read-only scan of both `/home/raw/Documents/workspace/caracalEcosystem/caracal` and `/home/raw/Documents/workspace/caracalEcosystem/caracalEnterprise`, here is a prioritized legacy/dead code removal list:

## **PRIORITIZED LEGACY CODE REMOVAL LIST**

### **TIER 1: CRITICAL - Must Remove (High Risk, Blocking Hard-Cut)**

| File | Function/Code | Reason | Risk | Action |
|------|---|---|---|---|
| [setup.py](setup.py) | Entire file | Legacy build tool compatibility shim; marked as "exists for compatibility with older build tools" | **HIGH** - Breaks package distribution if kept alongside modern pyproject.toml only | **REWRITE** - Remove entirely; rely solely on pyproject.toml |
| [packages/caracal/caracal/runtime/hardcut_preflight.py](packages/caracal/caracal/runtime/hardcut_preflight.py#L73-L85) | `_FORBIDDEN_COMPAT_ENV_VARS` constant + `_compatibility_violations()` function | Detects and forbids legacy compatibility environment variables: `CCL_COMPAT_ON`, `CCL_COMPAT_ALIASES`, `CCL_COMPAT_MODE`, `CCL_DUAL_WRITE_ON`, `CCL_DUAL_WRITE_WINDOW` | **CRITICAL** - These vars enable dual-write and fallback paths that violate hard-cut guarantees; preflight already rejects them | **PATCH** - Remove detection logic and forbid at bootstrap; code is dead once deployment blocks these vars |
| [packages/caracal-server/caracal/deployment/config_manager.py](packages/caracal-server/caracal/deployment/config_manager.py#L390-L425) | `workspace_db_restore_compatibility_fallback` log marker + fallback SQL sanitization in `_restore_workspace()` | Sanitizes SQL dumps to remove unsupported legacy settings when restoring workspace DBs; fallback handler catches restoration errors | **HIGH** - Masks database schema violations; allows stale configs to persist; should fail loudly instead | **REWRITE** - Replace with strict validation; fail if legacy markers found rather than silently strippping them |
| [packages/caracal-server/caracal/redis/client.py](packages/caracal-server/caracal/redis/client.py#L127-L150) | `getdel()` method with Lua fallback | Falls back to Lua script when Redis `GETDEL` command unavailable (Redis < 6.2) | **MEDIUM** - Hard-cut requires Redis 6.2+; fallback supports outdated versions | **PATCH** - Replace with direct `GETDEL` call; ensure minimum Redis version enforced at deployment validation |

---

### **TIER 2: HIGH - Remove in Current Release Phase**

| File | Function/Code | Reason | Risk | Action |
|------|---|---|---|---|
| [db/migrations/versions/l1m2n3o4p5q6_principal_key_custody_hardcut.py](packages/caracal-server/caracal/db/migrations/versions/l1m2n3o4p5q6_principal_key_custody_hardcut.py#L68-201) | AWS KMS key custody migration logic | Migrates AWS KMS-backed principal keys to Vault; reads & converts `aws_kms_ciphertext_b64`, `aws_kms_key_id`, `aws_kms_region` metadata; creates `principal_key_custody_aws_kms` table then drops it | **MEDIUM** - AWS KMS support is being dropped; migration code is read-only after execution; migration file itself is obsolete post-hard-cut | **PATCH** - Leave migration as-is for existing deployments; rename migration file to mark as "archived_aws_kms_.*" to signal removal after grace period |
| [db/migrations/versions/n3o4p5q6r7s8_authority_relational_constraints_hardcut.py](packages/caracal-server/caracal/db/migrations/versions/n3o4p5q6r7s8_authority_relational_constraints_hardcut.py#L150-156) | AWS KMS backend detection in authority migration | Detects if principal was using AWS KMS backend via `backend = 'aws_kms'` predicate | **MEDIUM** - Only runs once during migration; harmless but evidence of legacy support | **PATCH** - Remove AWS KMS branch; assume all principals migrated to Vault |
| [db/migrations/versions/s8t9u0v1w2x3_drop_sync_state_tables_hardcut.py](packages/caracal-server/caracal/db/migrations/versions/s8t9u0v1w2x3_drop_sync_state_tables_hardcut.py) | Complete migration file | Drops legacy sync-state ORM tables (`sync_operations`, `sync_conflicts`, `sync_metadata`) after SyncEngine decoupling | **LOW** - Migration is one-time only; purely structural cleanup; no active code paths | **PATCH** - Keep as-is; archive filename to mark as "legacy_cleanup_.*" for documentation |
| [sdk/python-sdk/src/caracal_sdk/_compat.py](sdk/python-sdk/src/caracal_sdk/_compat.py) | Entire file | Internal SDK compatibility helpers; currently only exports standard exception types and `get_logger()`, `get_version()` | **MEDIUM** - Filename suggests hidden legacy shim logic; tests check it has no "fallback alias logic" (line 1492 in test_hardcut_release_gates.py) | **PATCH** - Rename to `_common.py` or `_internal.py` to remove "compat" connotation; ensure no conditional imports or version checks exist |
| [src/lib/permissions.ts](caracalEnterprise/src/lib/permissions.ts#L92-96) | `permissionSnapshot()` defaultRole fallback chain | When organization permissions snapshot is stale/missing, defaults to `defaultRoles[2]` (generic fallback) for undefined pages; applies fallback permissions with three-level OR: `role.permissions?.[page] \|\| fallback.permissions[page] \|\| { view: false, edit: false, lock: false }` | **MEDIUM** - Silently degrades permission to defaults; can mask misconfiguration; should fail loudly in hard-cut | **REWRITE** - Remove fallback chain; throw error if organization permissions undefined instead of defaulting |

---

### **TIER 3: MEDIUM - Schedule for Next Phase**

| File | Function/Code | Reason | Risk | Action |
|------|---|---|---|---|
| [src/lib/api.ts](caracalEnterprise/src/lib/api.ts#L665-667) | `regenerate(oldLicenseKey)` parameter | API endpoint accepts `old_license_key` field for license regeneration; legacy field from password-era auth | **LOW** - Unused if frontend doesn't send old key; but parameter still recognized | **PATCH** - Deprecate in endpoint; reject with clear error message advising new flow |
| [services/api/src/caracal_api/routes/license.py](caracalEnterprise/services/api/src/caracal_api/routes/license.py#L113-394) | `LicenseRegenerateRequest.old_license_key` field + validation at line 394 | Validates that submitted `old_license_key` matches current license before regeneration; legacy security model | **LOW** - Validation is defensive; no fallback; but indicates older auth pattern still supported | **PATCH** - Remove field validation; require only new license data path |

---

### **TIER 4: DOCUMENTATION ONLY (Intentional Legacy Markers)**

These are **NOT dead code** — they are intentional surveillance/validation markers that tests enforce *stay absent*:

| File | Pattern | Purpose | Status |
|------|---|---|---|
| [scripts/hardcut_forbidden_marker_scan.py](scripts/hardcut_forbidden_marker_scan.py) | Marker definitions (lines 25-132) | Defines 17 forbidden legacy patterns (AWS KMS, Fernet, keyring, boto3, Infisical endpoints, sync tables, etc.) | **ACTIVE** — Scans all code; fails CI if any marker found |
| [packages/caracal/caracal/runtime/hardcut_preflight.py](packages/caracal/caracal/runtime/hardcut_preflight.py#L95-115) | `_FORBIDDEN_CONFIG_MARKERS` | Runtime checks that legacy config markers absent from deployment files | **ACTIVE** — Enforced at startup |
| [tests/unit/runtime/test_hardcut_release_gates.py](tests/unit/runtime/test_hardcut_release_gates.py) | 15+ test functions validating absence of legacy patterns | Ensures boto3, Fernet, keyring, AWS KMS, Infisical, sync modules never reappear | **ACTIVE** — CI gate |

---

## **SUMMARY TABLE**

| Pattern | Status | Files Affected | AI Reuse Risk | Notes |
|---------|--------|-----------------|---------------|-------|
| AWS KMS/Fernet/keyring/boto3 imports | **REMOVED** | Migration files only; no active imports | **LOW** — Only in archived migrations | Search pattern in forbidden marker scan prevents re-introduction |
| Sync-state tables & sync engine | **REMOVED** | Migration files; no ORM models; no CLI commands | **CRITICAL** — Tests actively verify removal | Hard-cut blocker once deployed |
| Compatibility env vars (CCL_COMPAT_*) | **BLOCKED** | config_manager.py (sanitizer only); hardcut_preflight.py (enforcer) | **MEDIUM** — Preflight rejects but config_manager silently strips | Replace config fallback with strict validation |
| License password auth | **DEPRECATED** | license.py (api.ts); old_license_key parameter | **LOW** — Parameter exists but flow updated | Remove in next release |
| Setup.py | **LEGACY** | Root setup.py only | **LOW** — Only for distribution | Safe to delete if pyproject.toml handles all build cases |
| Redis GETDEL fallback | **OUTDATED VERSION CHECK** | redis/client.py | **MEDIUM** — Supports Redis < 6.2 | Enforce Redis 6.2+ minimum at validation |
| Permission defaulting chain | **SILENT DEGRADATION** | permissions.ts | **HIGH** — Can hide config errors | Replace with loud failure |

---

## **REWRITE VS. PATCH DECISION**

| Action | Files | Rationale |
|--------|-------|-----------|
| **DELETE** | `setup.py` | Complete legacy artifact; pyproject.toml is mandatory replacement |
| **REWRITE** | [config_manager.py](packages/caracal-server/caracal/deployment/config_manager.py)::workspace_db_restore_compatibility_fallback | Replace silent fallback SQL sanitization with strict schema validation |
| **REWRITE** | [permissions.ts](caracalEnterprise/src/lib/permissions.ts)::permissionSnapshot() | Remove multi-level fallback chain; throw on invalid state |
| **PATCH** | Redis GETDEL, License endpoints, preflight env var checks | Minimal changes; already isolated; no dependency trees |
| **REMOVE** | `_FORBIDDEN_COMPAT_ENV_VARS` logic in hardcut_preflight | Dead once deployment blocks these vars; preflight becomes noop |
| **KEEP (archive)** | Migration files | One-time use; beneficial audit trail; rename to mark archived |
| **RENAME** | `_compat.py` | Reduce misleading "compat" signal; actual code has no fallback logic |

---

## **MISSED OPPORTUNITIES FOR AI MISUSE**

1. **config_manager.py fallback logic** — An AI copying SQL sanitization code for "robustness" would perpetuate silent failures instead of strict validation
2. **permissions.ts fallback chain** — AI might expand the OR-chain with more fallback tiers, making permission bugs harder to debug  
3. **Redis getdel() Lua fallback** — AI might backport this to support older Redis versions, re-introducing version compatibility code that hard-cut forbids
4. **License old_license_key parameter** — AI might invent new uses for the deprecated field instead of removing it cleanly

---

**No edits applied per your read-only request. All findings are analysis-ready for implementation in the refactoring phase.**