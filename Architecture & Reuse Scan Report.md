## Architecture & Reuse Scan Report

**Scope**: `/caracal` and `/caracalEnterprise` (excluding docs, code generation, and commentary)

---

### **1. CRITICAL FINDINGS: Large, Brittle Monolithic Files**

#### [caracal/packages/caracal-server/caracal/runtime/entrypoints.py](caracal/packages/caracal-server/caracal/runtime/entrypoints.py) — **REWRITE**
- **Size**: ~1,200 lines (400 functions + deep call chains)
- **Brittle Core Issues**:
  - Monolithic CLI entrypoint mixing Docker resource management, database backups, vault operations, and TUI orchestration
  - Duplicated Docker inspection patterns: `_inspect_container_labels()`, `_inspect_container_image()`, `_list_network_container_names()` contain identical retry/error-handling logic repeated 4+ times
  - Tight coupling: CLI handlers directly call subprocess, Docker API, environment variable reads without abstraction
  - No error boundaries: All failures in resource cleanup propagate; no graceful partial-cleanup on error
- **Risk**: Single point of failure for runtime management; any CLI flag change touches 200+ lines
- **Recommendation**: **Extract Docker management layer** into `caracal/runtime/docker_manager.py` with:
  - `DockerResourceManager` class pooling container/volume/network inspection
  - Reusable retry/error-handling middleware
  - Separate `TUIOrchestrator` for Flow launch
  - **Decision**: Full **REWRITE** — current structure is too tangled for safe patching

#### [caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py](caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py) — **REWRITE**
- **Size**: Over 3,500 lines (evident from search results showing line 3530-3558)
- **Tight Coupling Issues**:
  - `EnterpriseCaracalClient` wraps *every* core operation (`validate_workspace_membership()`, `verify_intent()`, `validate_mandate_with_details()`) with workspace scoping
  - Creates its own SQLAlchemy engine/session despite caracal-server already having `DatabaseConnectionManager`
  - Re-implements mandate/authority logic; tight dependency on `caracal.core.mandate.MandateManager` without adapter
  - Business logic mixed with database session management at every call site
- **Risk**: Changes to core mandate validation cascade through all enterprise routes that instantiate this client
- **Recommendation**: **Decompose into three layers**:
  1. **Thin adapter**: Workspace scoping only (`WorkspaceScopedCaracalBridge`)
  2. **Reuse from core**: Direct use of `MandateManager` (already stable)
  3. Request/response translation separate from session management
- **Decision**: Full **REWRITE** with dependency injection of session factory

#### [caracalEnterprise/services/api/src/caracal_api/middleware/auth.py](caracalEnterprise/services/api/src/caracal_api/middleware/auth.py) — **PATCH**
- **Lines**: ~260
- **Acceptable Issues**:
  - Token claim extraction duplicated from [caracalEnterprise/src/lib/auth.ts](caracalEnterprise/src/lib/auth.ts#L30-L50) client-side
  - Fallback token extraction logic is defensive but redundant
- **Why Patch**: Middleware is isolated, no downstream consumers rely on its internal details
- **Action**: Extract token payload decoding into shared utility `_extract_token_claims()` that both client and server import

---

### **2. DUPLICATION ANALYSIS**

#### **A. Authentication & Token Management — UNSAFE REUSE PATTERN**

**Files Involved**: (8 copies of similar token handling)
- [caracal/packages/caracal/runtime/environment.py](caracal/packages/caracal/runtime/environment.py#L30-L40): Hardcoded mode aliases (`MODE_DEV`, `MODE_STAGING`, etc.)
- [caracalEnterprise/services/api/src/caracal_api/config.py](caracalEnterprise/services/api/src/caracal_api/config.py#L32-L45): Enterprise-specific env map + defaults
- [caracalEnterprise/services/api/src/caracal_api/utils/auth.py](caracalEnterprise/services/api/src/caracal_api/utils/auth.py#L24-L110): Session key reference parsing, key-pair bootstrapping
- [caracalEnterprise/services/gateway/auth.py](caracalEnterprise/services/gateway/auth.py): Separate `Authenticator` class (mTLS + JWT + API key)
- [caracalEnterprise/src/lib/auth.ts](caracalEnterprise/src/lib/auth.ts#L65-L90): Client-side token extraction from localStorage and JWT decoding

**Unsafe Pattern**: 
- Each service independently decodes JWT payloads and extracts workspace ID (`workspace_id`, `org`, `organization_id`)
- No central claim schema — enterprise API accepts multiple field names
- Token revocation via redis denylist in enterprise, but Core uses Caracal vault — no coordination

**Recommendation**: 
1. **Extract shared JWT utilities** into `caracal-shared-auth` package:
   - Canonical claim schema definition
   - Paymentload validation with field-name normalization
   - Denylist adapter for both Redis and vault backends
2. **Patch enterprise middleware**: Import from shared package instead of reimplementing
3. **Risk Reduction**: Single source of truth for token structure

---

#### **B. Secret Management — CONTROLLED DUPLICATION (Acceptable)**

**Backend Implementations** (all in separate services for isolation):
- [caracal/packages/caracal-server/caracal/core/vault.py](caracal/packages/caracal-server/caracal/core/vault.py#L700-L1200): `CaracalVault` (main vault client)
- [caracalEnterprise/services/gateway/secret_manager.py](caracalEnterprise/services/gateway/secret_manager.py): `SecretManager` + `CaracalVaultBackend` 
- [caracal/packages/caracal-server/caracal/provider/credential_store.py](caracal/packages/caracal-server/caracal/provider/credential_store.py): workspace-scoped credential wrapping
- [caracalEnterprise/services/api/src/caracal_api/routes/secrets.py](caracalEnterprise/services/api/src/caracal_api/routes/secrets.py#L1-L280): workspace secret routes

**Why This Is OK**:
- Each layer has a different _responsibility_ (not duplication):
  - Core Vault: Direct secret API calls with retry logic
  - Gateway Secret Manager: Caching + tier-based backend selection
  - Provider credentials: Workspace-scoped locking + migration helpers
  - API routes: HTTP validation + permission enforcement
- All delegate to `get_vault()` singleton
- **No unsafe reuse**: Each knows its boundaries

**Action**: **No change needed** — this is intentional layering.

---

#### **C. Configuration Loading — UNSAFE PATTERN**

**Files**:
- [caracal/packages/caracal/runtime/environment.py](caracal/packages/caracal/runtime/environment.py): Runtime mode resolution
- [caracalEnterprise/services/api/src/caracal_api/config.py](caracalEnterprise/services/api/src/caracal_api/config.py#L14-L80): Pydantic-based settings + legacy env var mapping
- [caracalEnterprise/services/vault/app.py](caracalEnterprise/services/vault/app.py#L260-L300): Direct `os.getenv()` + hardcoded defaults

**Problem**:
- No shared configuration schema
- Enterprise manually maps `CCLE_*` to `CCL_*` with hardcoded defaults
- Vault service uses raw `os.getenv()` with no validation
- Three separate places checking `CCL_VAULT_*` environment variables

**Recommendation**:
1. **Create `caracal-config` shared module** with:
   - `EnvResolver` class handling CCL_* env var hierarchy
   - Canonical defaults (vault URL, signing algorithm, etc.)
   - Validation middleware
2. **Each service imports**: `from caracal_config import ConfigResolver`
3. **Patch**: All three services to use single resolver

---

### **3. TIGHT COUPLING ISSUES**

#### **Gateway ↔ Enterprise API**
**File**: [caracalEnterprise/services/gateway/__init__.py](caracalEnterprise/services/gateway/__init__.py) — lazy export module
- Current issue: Gateway exports 50+ symbols via `__getattr__()` magic — makes IDE support impossible, runtime errors if symbol name mispelled
- **Recommendation**: Convert to explicit imports; split into `gateway.auth`, `gateway.cache`, `gateway.proxy`, etc.
- **Decision**: **PATCH** — Replace magic exports with explicit `from .auth import *` chains

#### **Enterprise API ↔ Caracal Core Models**
- Enterprise routes directly query `caracal.db.models.Principal`, `ExecutionMandate`, `AuthorityPolicy`
- Core changes to schema (e.g., renaming `owner` → `workspace_id`) will break all enterprise routes
- **Solution**: Create adapter layer `caracal_api/adapters/core_models.py` that maps enterprise domain objects to core models
- **Decision**: **PATCH** — Introduce adapter, update 5-10 route files to use it

#### **Enterprise API ↔ Vault Initialization**
[caracalEnterprise/services/api/src/caracal_api/main.py](caracalEnterprise/services/api/src/caracal_api/main.py#L25-L40) imports `caracal.runtime.hardcut_preflight` → depends on caracal-server startup concerns
- Couples FastAPI startup to Caracal runtime lifecycle checks
- **Solution**: Move `assert_enterprise_hardcut()` into enterprise module, or expose as pluggable validator
- **Decision**: **PATCH** — Extract to separate validation module

---

### **4. REQUIRED REWRITES (Loss if not done)**

| File | Lines | Reason | Impact |
|------|-------|--------|--------|
| [caracal/packages/caracal-server/caracal/runtime/entrypoints.py](caracal/packages/caracal-server/caracal/runtime/entrypoints.py) | 1,200 | Too many responsibilities, duplicated Docker logic, tight subprocess coupling | CLI reliability, testability |
| [caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py](caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py) | 3,500+ | Monolithic workspace-scoped wrapper, recreates DB sessions, tight mandate coupling | Route maintainability, core schema changes |
| [caracalEnterprise/services/api/src/caracal_api/config.py](caracalEnterprise/services/api/src/caracal_api/config.py) (Settings class) | 250+ | Manual env var mapping, legacy compatibility aliases, no shared schema | Config evolution, deployment flexibility |

---

### **5. SAFE EXTRACTION OPPORTUNITIES**

**Extract `caracal-vault-adapter` (Low Risk)**
- Location: New package `caracal/packages/caracal-vault-adapter/`
- Move from `caracal/packages/caracal-server/caracal/core/vault.py`:
  - `CaracalVault` class definition
  - Retry logic (`_RETRYABLE_STATUS_CODES`, `_request()` method)
  - HTTP response parsing (`_extract_secret_value()`, `_extract_secret_names()`)
- Keep in caracal-server:
  - Database schema (still coupled to Core)
  - Principal integration (Caracal-specific)
- **Benefit**: Gateway and enterprise can import without pulling entire caracal-server
- **Effort**: ~400 lines to extract

**Extract `caracal-db` as separate package (Medium Risk)**
- Location: `caracal/packages/caracal-db/`
- Move from caracal-server:
  - [caracal/packages/caracal-server/caracal/db/models.py](caracal/packages/caracal-server/caracal/db/models.py): Principal, ExecutionMandate, LedgerEvent
  - [caracal/packages/caracal-server/caracal/db/connection.py](caracal/packages/caracal-server/caracal/db/connection.py): Connection pooling
- Keep in caracal-server:
  - Query builders (business logic)
  - Repository interfaces
- **Benefit**: Enterprise directly uses models without importing all of caracal-server
- **Risk**: Schema migrations must land in both packages first (currently in enterprise/alembic/)
- **Effort**: ~800 lines to extract

---

### **6. Error Handling Duplication (PATCH)**

**Pattern Found Across Services**:
- Each service defines own error hierarchy:
  - Caracal: `VaultError`, `VaultUnavailableError`, `SecretNotFound` 
  - Enterprise: `CaracalError`, `AuthorityError`, `PolicyViolationError`
  - Gateway: `SecretManagerError`, `NullBackendError`
- No error-code standardization

**Recommendation**: **Shared error module** `caracal/packages/caracal-common/errors.py`:
```python
class CaracalErrorCode(Enum):
    VAULT_UNAVAILABLE = "vault_unavailable"
    SECRET_NOT_FOUND = "secret_not_found"
    POLICY_VIOLATION = "policy_violation"
    INVALID_MANDATE = "invalid_mandate"

class CaracalError(Exception):
    def __init__(self, code: CaracalErrorCode, detail: str):
        self.code = code
        self.detail = detail
```
- **Decision**: **PATCH** — Create shared module, replace 15-20 custom error classes

---

### **7. Database Session Management — MILD DUPLICATION**

**Files**:
- [caracal/packages/caracal-server/caracal/db/connection.py](caracal/packages/caracal-server/caracal/db/connection.py#L220-L390): Full-featured `DatabaseConnectionManager`
- [caracalEnterprise/services/api/src/caracal_api/models/base.py](caracalEnterprise/services/api/src/caracal_api/models/base.py): Minimal setup (engine + SessionLocal)
- [caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py](caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py#L32-L50): Recreates SQLAlchemy engine in each client instance

**Issues**:
- Enterprise API duplicates connection setup instead of reusing Core's manager
- Each `EnterpriseCaracalClient` instance creates its own engine (connection pool leak)
- No health checks or schema version verification in enterprise

**Recommendation**: 
1. **Reuse** `caracal.db.connection.DatabaseConnectionManager` in enterprise API startup
2. **Patch** `EnterpriseCaracalClient` to accept session factory instead of creating its own:
   ```python
   class EnterpriseCaracalClient:
       def __init__(self, workspace_id: str, session_factory):
           self.session_factory = session_factory  # Inject
   ```
3. Update routes to instantiate client with `caracal.db.connection.get_connection_manager().get_session()`
- **Effort**: ~50 lines of patch work

---

### **8. Summary Table: Rewrite vs. Patch Decisions**

| Module | Type | Decision | Effort | Risk |
|--------|------|----------|--------|------|
| `caracal/runtime/entrypoints.py` | CLI orchestration | **REWRITE** | High | High |
| `caracalEnterprise/integrations/caracal_core.py` | Workspace wrapper | **REWRITE** | High | High |
| `caracalEnterprise/config.py` (Settings) | Config loading | **REWRITE** | Medium | Medium |
| Authentication token utils | Utilities | **PATCH** + extract shared module | Low | Low |
| Database session management | Connection pooling | **PATCH** + reuse | Low | Low |
| Error handling | Error classes | **PATCH** + shared module | Low | Low |
| Gateway exports | Module structure | **PATCH** | Low | Low |
| Vault integration | Secret storage | **No change** (accept duplication) | — | — |
| Secret management layers | Routes + backends | **No change** (intentional) | — | — |

---

### **9. Extraction Candidates (Reduces Risk)**

**High Priority**:
1. **`caracal-db` package** — Move Core database models/connection management
2. **`caracal-config` package** — Centralized environment variable + schema loading
3. **`caracal-errors` package** — Standardized error codes across services

**When to Extract**: Before making rewrite decisions; allows incremental adoption.

---

### **FILES REQUIRING ATTENTION**

| Risk Category | Red Flag Files | Action |
|---|---|---|
| **Untouchable High Complexity** | [caracal/packages/caracal-server/caracal/runtime/entrypoints.py](caracal/packages/caracal-server/caracal/runtime/entrypoints.py) | Full rewrite recommended |
| | [caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py](caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py) | Full rewrite recommended |
| **Tight Coupling** | [caracalEnterprise/services/api/src/caracal_api/config.py](caracalEnterprise/services/api/src/caracal_api/config.py) | Extract config schema, patch all references |
| | [caracalEnterprise/services/gateway/__init__.py](caracalEnterprise/services/gateway/__init__.py) | Replace magic exports with explicit imports |
| **Safe Duplication** | [caracal/packages/caracal-server/caracal/core/vault.py](caracal/packages/caracal-server/caracal/core/vault.py) + [caracalEnterprise/services/gateway/secret_manager.py](caracalEnterprise/services/gateway/secret_manager.py) | Accept; different responsibilities |