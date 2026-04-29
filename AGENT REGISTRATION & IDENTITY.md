---

## AGENT REGISTRATION & IDENTITY AUDIT REPORT

### 1. REGISTRATION FLOW DIAGRAM

```
┌─────────────────────────────────────────────────────────┐
│ PRINCIPAL REGISTRATION FLOW                             │
└─────────────────────────────────────────────────────────┘

1. Entry Points:
   ├─ IdentityService.register_principal() [identity/service.py:23]
   ├─ EnterpriseCaracalClient.register_principal() [caracal_core.py:2661]
   └─ CLI bootstrap [cli/bootstrap.py:133]

2. Core: PrincipalRegistry.register_principal() [core/identity.py:136]
   ├─ Validate principal_kind ∈ {HUMAN, ORCHESTRATOR, WORKER, SERVICE}
   ├─ Check for DUPLICATE NAME globally (raises DuplicatePrincipalNameError)
   │  └─ Query: session.query(Principal).filter_by(name=name).first()
   ├─ Create Principal row:
   │  ├─ principal_id (UUID, PK)
   │  ├─ name (UNIQUE, indexed)
   │  ├─ principal_kind
   │  ├─ owner (workspace_id string)
   │  ├─ lifecycle_status, attestation_status
   │  └─ source_principal_id (for delegation lineage)
   ├─ [if generate_keys=true]
   │  └─ generate_and_store_principal_keypair(principal_id)
   │     └─ Vault-only: vault.ensure_asymmetric_keypair(ES256)
   │     └─ Create PrincipalKeyCustody + PrincipalKeyCustodyVault records
   └─ Commit transaction
      └─ Return PrincipalIdentity

3. SPAWN variant: SpawnManager.spawn_principal() [spawn.py:70]
   ├─ Check for DUPLICATE NAME globally (raises DuplicatePrincipalNameError)
   ├─ Check idempotency via PrincipalWorkloadBinding
   │  └─ workload = f"{issuer_uuid}:{idempotency_key}"
   │  └─ binding_type = "spawn_idempotency"
   ├─ Create Principal (status=PENDING_ATTESTATION, attestation=PENDING)
   ├─ Generate keypair (same as above)
   ├─ Create TWO PrincipalWorkloadBindings:
   │  ├─ Idempotency marker: workload = "{issuer}:{key}"
   │  └─ Bootstrap marker: workload = "attest-bootstrap:{principal_id}"
   ├─ Issue attenuated ExecutionMandate:
   │  ├─ issuer_id → subject_id (new principal)
   │  ├─ narrow resource/action scope
   │  └─ context_tags contain spawn markers
   └─ Commit transaction
      └─ Return SpawnResult with principal_id, mandate_id, nonce
```

---

### 2. IDENTITY UNIQUENESS VERDICT

| Dimension | Status | Details | Risk |
|-----------|--------|---------|------|
| **principal_id (UUID)** | ✅ PASS | PRIMARY KEY, never reused | None |
| **principal.name** | ⚠️ PARTIAL FAIL | GLOBALLY UNIQUE, NOT scoped by workspace | HIGH |
| **public_key** | ✅ PASS | Stored per principal, Vault-backed | None |
| **PrincipalKeyCustody.principal_id** | ✅ PASS | UNIQUE=True, 1:1 relationship | None |
| **Mandate scoping** | ✅ PASS | By UUID (issuer_id, subject_id), not by name | None |

**CRITICAL FINDING**: Principal names are globally unique, creating potential collisions across workspaces or for parallel tasks within the same workspace.

---

### 3. CONCRETE BYPASS & COLLISION RISKS

#### **RISK 1: Global Name Uniqueness (HIGH)**

**File**: [caracal-server/caracal/db/models.py:278](caracal-server/caracal/db/models.py#L278)
```python
name = Column(String(255), unique=True, nullable=False, index=True)
```

**Issue**: Two independent workspaces cannot spawn agents with the same name.

**Example**:
- Workspace A spawns worker named `"search-worker-v1"` → succeeds
- Workspace B spawns worker named `"search-worker-v1"` → **FAILS** with `DuplicatePrincipalNameError`

**Bypass Scenario**: Two workers on the same task must have **different names**, e.g., `"task-123-worker-1"` and `"task-123-worker-2"`. If naming is auto-generated from task ID, collisions are unavoidable.

---

#### **RISK 2: Bootstrap Idempotency Violation (MEDIUM)**

**File**: [caracal-server/caracal/cli/bootstrap.py:148-155](caracal-server/caracal/cli/bootstrap.py#L148-L155)
```python
existing_count = session.query(Principal).count()
if existing_count > 0 and not force:
    system_principal = (
        session.query(Principal)
        .filter_by(principal_kind=PrincipalKind.SERVICE.value, name="system")
        .first()
    )
    if system_principal is None:
        system_principal = session.query(Principal).first()  # ⚠️ ANY principal!
```

**Issue**: If "system" principal doesn't exist but OTHER principals do, bootstrap reuses the first principal instead of creating a new one.

**Exploit**: Manually create a non-system principal before bootstrap runs → bootstrap will fail idempotency check and may reuse wrong principal.

---

#### **RISK 3: Workspace Isolation in Name Uniqueness (MEDIUM)**

**File**: [caracal-server/caracal/core/spawn.py:151-156](caracal-server/caracal/core/spawn.py#L151-L156)
```python
duplicate = (
    self.db_session.query(Principal)
    .filter(Principal.name == principal_name)
    .first()
)
if duplicate is not None:
    raise DuplicatePrincipalNameError(...)
```

**Issue**: Name uniqueness is NOT scoped to workspace. The spawn check queries globally.

**Impact**: 
- Cross-tenant name collisions will cause spurious failures
- Two independent tasks in different workspaces cannot use same worker name

---

#### **RISK 4: Mandate/Session NOT Scoped by Name (PASS)**

**File**: [caracal-server/caracal/core/authority.py:182-220](caracal-server/caracal/core/authority.py#L182-L220)
```python
candidate_mandates = (
    self.db_session.query(ExecutionMandate)
    .filter(
        ExecutionMandate.subject_id == caller_uuid,  # ✅ UUID, not name
        ExecutionMandate.revoked.is_(False),
        ExecutionMandate.valid_from <= evaluation_time,
        ExecutionMandate.valid_until >= evaluation_time,
    )
    .all()
)
```

**Verdict**: ✅ **PASS** — Mandates are correctly scoped by `subject_id` (UUID), not principal name. No permission confusion from name reuse.

---

### 4. PARALLEL WORKER DISTINGUISHABILITY

**Question**: If two worker agents are spawned for the same task, how does the system tell them apart?

**Answer**: **By unique principal_id UUID**

- Each spawn creates a distinct Principal row with unique `principal_id` and `name`
- Idempotency is tracked via `PrincipalWorkloadBinding(workload=f"{issuer_id}:{idempotency_key}")`
- **Two identical spawn calls → same worker (idempotent replay)**
- **Two different idempotency keys → different workers**

**Vulnerability**: If the same task spawns multiple workers with auto-generated names from task ID, they **cannot have the same name** due to global uniqueness. Must use unique name per spawn attempt.

**Example**:
```python
# Task 123 spawns two workers:
SpawnManager.spawn_principal(
    issuer_principal_id=orchestrator_id,
    principal_name="task-123-worker-1",  # ✅ Unique
    idempotency_key="spawn-task-123-worker-1"
)

SpawnManager.spawn_principal(
    issuer_principal_id=orchestrator_id,
    principal_name="task-123-worker-2",  # ✅ Must be different
    idempotency_key="spawn-task-123-worker-2"
)

# Both FAIL if names collide or are reused
```

---

### 5. REUSE/PERMISSION CONFUSION ANALYSIS

| Component | Scoped By | Status | Risk |
|-----------|-----------|--------|------|
| **ExecutionMandate** | `issuer_id`, `subject_id` (UUIDs) | ✅ PASS | No name-based confusion |
| **SessionManager tokens** | `principal_id` (UUID) | ✅ PASS | JWT claims carry UUID |
| **PrincipalCapabilityGrant** | `principal_id` FK | ✅ PASS | Per-principal, not per-name |
| **AuthorityPolicy** | `principal_id` FK | ✅ PASS | Tied to UUID, not name |
| **Authority validation** | `caller_principal_id` (UUID) | ✅ PASS | All lookups by UUID |
| **PrincipalWorkloadBinding** | `principal_id` FK, `workload` string | ⚠️ WATCH | Workload not unique; idempotency relies on naming convention |

**Verdict**: ✅ **Mandates and permissions are correctly scoped by UUID, not name.** No accidental permission transfer via name reuse.

---

### 6. BOOTSTRAP IDEMPOTENCY

#### OSS Caracal Bootstrap [cli/bootstrap.py:133-182]

**Status**: ⚠️ **WEAK**

- **Idempotency check**: `if existing_count > 0 and not --force: skip creation`
- **Failure mode**: If "system" principal is missing but other principals exist, bootstrap will reuse the first principal in the DB instead of the "system" principal
- **Fix**: Require explicit `principal_kind == SERVICE and name == "system"` match, or create a new system principal

#### Enterprise Admin Bootstrap

**Status**: ❌ **UNCLEAR** — No `bootstrap_enterprise_admin` function found

- `register_principal` in enterprise API requires `workspace_id=owner` parameter
- No visible stable enterprise user metadata link (e.g., via email or UUID from OIDC/SAML)
- **Recommendation** (from repo memory): Use stable enterprise user metadata (not display_name) to tie admin registration to account

---

### 7. WORKLOAD BINDING ANALYSIS

**File**: [caracal-server/caracal/db/models.py:398-416](caracal-server/caracal/db/models.py#L398-L416)

```python
class PrincipalWorkloadBinding(Base):
    binding_id = Column(BigInteger, primary_key=True, autoincrement=True)
    principal_id = Column(PG_UUID, ForeignKey("principals.principal_id", ondelete="CASCADE"), nullable=False)
    workload = Column(String(255), nullable=False)  # ⚠️ NOT unique
    binding_type = Column(String(50), nullable=False, default="workload")
    created_at = Column(DateTime, nullable=False)
```

**Binding Types Observed**:
- `"spawn_idempotency"` → workload = `"{issuer_uuid}:{idempotency_key}"`
- `"attestation_bootstrap"` → workload = `"attest-bootstrap:{principal_id}"`

**Verdict**: 
- ✅ **Workload binding DOES tie identity to spawn context** via issuer + idempotency key
- ⚠️ **Workload field is NOT unique** — multiple bindings can share same workload
- ⚠️ **Binding does NOT tie to task instance UUID** — only to spawn issuance context

**Risk**: If two spawns occur with the same idempotency key from the same issuer, they will resolve to the same worker (correct). But if spawn keys are poorly generated, collisions can occur.

---

### SUMMARY: REGISTRATION FLOW (ASCII)

```
register_principal()
  ├─ Validate kind
  ├─ Check name UNIQUE (globally) ← ⚠️ RISK: Blocks parallel workers
  ├─ Create Principal(id=UUID, name, owner=workspace_id)
  ├─ Generate ES256 keypair → Vault
  ├─ Create PrincipalKeyCustody (1:1)
  ├─ Create PrincipalKeyCustodyVault (vault reference)
  └─ Commit
     └─ Return PrincipalIdentity(principal_id, name, public_key)

spawn_principal() [variant for delegated agents]
  ├─ Check name UNIQUE (globally) ← ⚠️ RISK
  ├─ Check idempotency via PrincipalWorkloadBinding
  │  └─ If (issuer_id, idempotency_key) exists → return cached result
  ├─ [else] Create Principal + mandate + bindings (same as above)
  └─ Return SpawnResult(principal_id, mandate_id, nonce)
```

---

## RECOMMENDED FIXES

### 1. **Scope Principal Names by Workspace (HIGH PRIORITY)**

Change unique constraint from global to composite:
```sql
ALTER TABLE principals 
DROP CONSTRAINT principals_name_key;
CREATE UNIQUE INDEX ix_principals_workspace_name 
ON principals(owner, name);
```

Update [models.py:278](caracal-server/caracal/db/models.py#L278):
```python
# OLD:
name = Column(String(255), unique=True, nullable=False, index=True)

# NEW:
name = Column(String(255), nullable=False, index=True)
# Add composite index in migration
Index("ix_principals_owner_name", "owner", "name", unique=True)
```

Update spawn/register checks:
```python
# OLD:
if self.session.query(Principal).filter_by(name=name).first():
    raise DuplicatePrincipalNameError(...)

# NEW:
if self.session.query(Principal).filter_by(owner=owner, name=name).first():
    raise DuplicatePrincipalNameError(...)
```

---

### 2. **Strengthen Bootstrap Idempotency (MEDIUM)**

[cli/bootstrap.py:148-160](caracal-server/caracal/cli/bootstrap.py#L148-L160):
```python
# OLD:
system_principal = (
    session.query(Principal)
    .filter_by(principal_kind=PrincipalKind.SERVICE.value, name="system")
    .first()
)
if system_principal is None:
    system_principal = session.query(Principal).first()  # ⚠️ WRONG

# NEW:
system_principal = (
    session.query(Principal)
    .filter_by(principal_kind=PrincipalKind.SERVICE.value, name="system")
    .first()
)
if system_principal is None:
    raise RuntimeError(
        "System principal 'system' not found. Bootstrap must find or create only 'system'. "
        "Pass --force to recreate."
    )
```

---

### 3. **Enterprise Admin Bootstrap: Stable Identity Link (MEDIUM)**

Define idempotency by enterprise user UUID (not display_name):
```python
def register_enterprise_admin(workspace_id: str, enterprise_user_uuid: str, ...):
    # Look up existing by stable enterprise_user_uuid in metadata
    principal = session.query(Principal).filter(
        Principal.owner == workspace_id,
        Principal.principal_metadata["enterprise_user_uuid"].astext == enterprise_user_uuid
    ).first()
    
    if principal:
        return principal  # Idempotent replay
    
    # Create new principal
    name = f"admin-{enterprise_user_uuid[:8]}"  # Tie name to UUID
    return PrincipalRegistry(...).register_principal(
        name=name,
        metadata={"enterprise_user_uuid": enterprise_user_uuid},
        ...
    )
```

---

### 4. **Add Task-Instance Binding (OPTIONAL, for future auditing)**

If true task-per-worker isolation is required:
```python
class Principal(Base):
    task_instance_id = Column(String(255), nullable=True, index=True)  # Optional
    # Composite: (owner, task_instance_id, principal_kind) for scoping workers to task

# In spawn:
# task_instance_id = f"{task_id}:{uuid4()}"  # Unique per spawn
```

---

## FINAL VERDICT

| Category | Status | Confidence |
|----------|--------|------------|
| **Principal ID Uniqueness** | ✅ PASS | 100% |
| **Principal Name Uniqueness** | ⚠️ FAIL | 100% — Global, not workspace-scoped |
| **Mandate Scoping by ID** | ✅ PASS | 100% |
| **Session/Token Scoping** | ✅ PASS | 100% |
| **Permission Confusion** | ✅ PASS | 100% — All lookups by UUID |
| **Idempotency (Spawn)** | ✅ PASS | 95% — Workload binding works, but names must be unique |
| **Bootstrap Idempotency** | ⚠️ WEAK | 90% — OSS bootstrap has edge case; enterprise needs metadata tie-in |
| **Parallel Workers Distinguishability** | ✅ PASS | 100% — By principal_id UUID, but names must differ |

**Overall Risk**: **MEDIUM** — Principal name global uniqueness will cause cross-workspace and parallel-task failures. Recommend urgent fix #1 above.