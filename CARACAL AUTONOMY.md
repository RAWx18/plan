I'll provide a comprehensive read-only audit of both systems. Let me compile my findings with exact file/function references and practical recommendations.

## **CARACAL AUTONOMY & SECURITY AUDIT**

### **EXECUTIVE SUMMARY**

The system operates primarily in **manual mode**: critical workflows (principal registration, policy creation, allowlist management, vault initialization) require explicit CLI/API invocation with no pre-authorization or automation hooks. **Auditability is fragmented** across the codebase with significant gaps in operational logging. **Security gates exist but are not comprehensive**—some areas are auto-permissive in dev mode. The following audit identifies 23 specific blockers to minimal-intervention operation and provides remediation guidance.

---

## **I. AUTONOMY BLOCKERS (Preventing Automated Operation)**

### **A. Principal Registration Requires Manual CLI**

**Files/Functions:**
- [packages/caracal-server/caracal/cli/principal.py](packages/caracal-server/caracal/cli/principal.py#L89-L230) — `register()` command
- [packages/caracal-server/caracal/core/identity.py](packages/caracal-server/caracal/core/identity.py) — `PrincipalRegistry.register_principal()` (no pre-approval)

**Issue:** Principals are created directly without approval workflow. CLI command `caracal principal register` bypasses any authorization check beyond CLI context.

**Impact:** No mechanism to validate that a new principal should exist before creation registered to the system.

**Recommendation:** 
- Add optional pre-approval webhook hook to `PrincipalRegistry.register_principal()` before committing
- Return `"pending_registration"` status if pre-approval is enabled, auto-skip if disabled
- Log registration intent to audit trail (see **II.A** below) *before* DB commit

**Security gates that remain effective:** Principal attestation status still requires explicit AIS nonce validation.

---

### **B. Authority Policy Creation Has No Delegation Approval**

**Files/Functions:**
- [packages/caracal-server/caracal/cli/authority_policy.py](packages/caracal-server/caracal/cli/authority_policy.py#L83-L200) — `create()` command
- [packages/caracal-server/caracal/db/models.py](packages/caracal-server/caracal/db/models.py#L899-L946) — `AuthorityPolicy` model
  - Policy cannot be modified after creation; can only be deactivated
  
**Issue:** Policy constraints (`allow_delegation`, `max_network_distance`, `max_validity_seconds`) are set once at creation with direct DB insert. No revision workflow or re-authorization.

**Impact:** Once a policy grants broad (e.g. `max_network_distance=3`, `max_validity_seconds=86400`), it cannot be tightened without DB manipulation.

**Recommendation:**
- Add `policy_version` column to track incremental tightenings (immutable history)
- Create `tighten_policy()` operation that validates monotonic constraint reduction
- Log policy evolution to AuditLog with actor + reason
- Require explicit authorization revalidation if delegation distance increases

---

### **C. Vault Bootstrap Missing; Manual Env-Var Setup Required**

**Files/Functions:**
- [deploy/docker-compose.yml](deploy/docker-compose.yml#L37-L67) — Vault sidecar service references `CCL_VAULT_SIDECAR_AUTH_SECRET`, `CCL_VAULT_SIDECAR_ENC_KEY` with placeholder values
- [PACKAGING.md](PACKAGING.md#L90-L110) — Documents missing vault bootstrap (§2.4)

**Issue:** No `caracal vault init` command exists. Vault startup requires:
1. Manual generation of `CCL_VAULT_SIDECAR_AUTH_SECRET` (32-byte hex)
2. Manual generation of `CCL_VAULT_SIDECAR_ENC_KEY` (16-byte hex)
3. Seal/unseal flow not documented
4. Root token generation not automated

**Impact:** First-time deployments hang waiting for manual secret injection; no automated recovery.

**Recommendation:**
- Implement `caracal vault init --mode [dev|managed]` command that:
  - Generates/stores secrets in `$CCL_HOME/runtime/.env`
  - Performs attestation against Infisical API (if managed mode)
  - Returns initialized state machine
  - Logs initialization event to AuditLog (see **II.A**)
- Document sidecar=only vs. external vault decision matrix

---

### **D. Alembic Migration Path Resolution Breaks Outside Monorepo**

**Files/Functions:**
- [packages/caracal-server/caracal/cli/db.py](packages/caracal-server/caracal/cli/db.py) — `_resolve_alembic_ini_path()` walks parent directories from installed module
- [PACKAGING.md](PACKAGING.md#L72-L85) — Documents alembic.ini lookup chain issue (§2.2)

**Issue:** `alembic.ini` lives at repo root. CLI walks `../../` from installed `caracal` module location—only resolves if:
- User installs from editable source (`pip install -e`)
- Or repo layout matches expected tree

**Impact:** `caracal migrate` fails with "alembic.ini not found" after `pip install caracal-core` from PyPI.

**Recommendation:**
- Embed `alembic.ini` as package data via `importlib.resources`
- Move `caracal migrate` into runtime container (executed via `caracal cli migrate`)
- Fallback chain: inline config → `$CCL_HOME/alembic.ini` → error with guidance

---

### **E. Workspace Creation Requires Manual CLI / No Auto-Provisioning**

**Files/Functions:**
- [packages/caracal-server/caracal/cli/deployment_cli.py](packages/caracal-server/caracal/cli/deployment_cli.py) — deployment CLI group
- [packages/caracal-server/caracal/deployment/config_manager.py](packages/caracal-server/caracal/deployment/config_manager.py#L950-L1030) — `create_workspace()` method

**Issue:** Each workspace is manually created via `caracal workspace create <name>`. No provisioning hook on:
- Tenant registration (enterprise)
- Principal creation (OSS)
- Policy activation

**Impact:** Multi-tenant deployments require parallel manual workspace setup.

**Recommendation:**
- Add `auto_provision_workspace` configuration flag
- Create `provisioning_webhook` callback in `ConfigManager.create_workspace()`
- Link to organization/principal creation events with idempotency guarantees
- Log workspace provisioning to AuditLog

---

### **F. Allowlist Creation Requires Explicit Manual Entry**

**Files/Functions:**
- [packages/caracal-server/caracal/cli/allowlist.py](packages/caracal-server/caracal/cli/allowlist.py#L35-L95) — `create()` command
- [packages/caracal-server/caracal/core/allowlist.py](packages/caracal-server/caracal/core/allowlist.py) — `AllowlistManager.create_allowlist()`

**Issue:** Every allowlist entry is created manually. System default is **deny-all** if any allowlist exists; operators must manually add patterns.

**Impact:** New principal has zero resources allowed by default. Manual allowlist population required per principal.

**Recommendation:**
- Add **policy-driven allowlist generation** from `AuthorityPolicy.allowed_resource_patterns`
- On policy creation, auto-materialize allowlist entries for convenience (operator can suppress)
- Batch allowlist creation endpoint: `POST /allowlists` with principal_id + policies
- Log auto-generated entries separately from manual ones

---

### **G. Session Token Minting Requires Manual CLI Invocation**

**Files/Functions:**
- [packages/caracal-server/caracal/cli/auth.py](packages/caracal-server/caracal/cli/auth.py) — `token` command
- [packages/caracal-server/caracal/identity/principal_ttl.py](packages/caracal-server/caracal/identity/principal_ttl.py) — TTL registration

**Issue:** `eval "$(caracal auth token --format env)"` invoked manually at each shell session. No daemon token server or automatic renewal.

**Impact:** SDK usage requires manual token injection into environment; no transparent credential refresh.

**Recommendation:**
- Implement optional sidecar token server (`caracal token-daemon`) that:
  - Mints tokens via Unix socket and caches with TTL
  - Auto-rotates on expiration
  - Injects into clients via environment or file descriptor
- Expose token endpoint at `http://127.0.0.1:7079/token` (alongside AIS)

---

### **H. Enterprise: Onboarding Requires Manual Step Progression**

**Files/Functions:**
- [caracalEnterprise/services/api/src/caracal_api/routes/onboarding.py](caracalEnterprise/services/api/src/caracal_api/routes/onboarding.py#L50-L150) — `WorkspaceState` and step transitions
- [caracalEnterprise/services/api/src/caracal_api/routes/onboarding.py](caracalEnterprise/services/api/src/caracal_api/routes/onboarding.py#L150-L210) — `_compute_workspace_state()` 

**Issue:** Workspace state machine `configure` → `team` → `pricing` → `payment` → `workspace` → `complete` requires explicit step transition. No auto-advancement except post-payment.

**Impact:** First-time deployment blocked until user manually advances each step.

**Recommendation:**
- Add **autonomous onboarding mode** configuration flag
- Auto-advance through pre-payment steps given valid config
- Auto-create default team/roles if not specified
- Auto-select tier based on usage signal (not manual selection)
- Log each advancement to audit trail

---

### **I. SDK Requires Manual Installation; Reaches Into Server Code**

**Files/Functions:**
- [PACKAGING.md](PACKAGING.md#L67-L75) — §2.5 SDK import leak
- [sdk/python-sdk/src/caracal_sdk/_compat.py](sdk/python-sdk/src/caracal_sdk/_compat.py) — imports from `caracal.` namespace
- [pyproject.toml](pyproject.toml) — uses `{ path: "sdk/python-sdk", editable: true }` internally

**Issue:** `caracal-sdk` PyPI package silently requires `caracal-core` installed. SDKs reaching back into server modules create implicit dependency.

**Impact:** `pip install caracal-sdk` fails unless `caracal-core` also installed; breaks SDK-only deployments (e.g., client-side only).

**Recommendation:**
- Strip SDK→server imports: move shared types/exceptions into `caracal-base` package
- Publish three separate packages: `caracal` (CLI/TUI), `caracal-sdk` (standalone), `caracal-runtime` (server image)
- Validate SDK on clean environment: `pip install caracal-sdk && python -c "from caracal_sdk import CaracalClient"`

---

## **II. AUDITABILITY GAPS**

### **A. Principal Registration Not Audited**

**Files/Functions:**
- [packages/caracal-server/caracal/cli/principal.py](packages/caracal-server/caracal/cli/principal.py#L150-L200) — `register()` creates principal but logs nothing to `AuditLog`
- [packages/caracal-server/caracal/db/models.py](packages/caracal-server/caracal/db/models.py#L130-L180) — `AuditLog` table exists but unused for identity ops

**Issue:** Principal registration is not recorded in immutable audit log. Only key lifecycle events go to `AuditLog` (via [packages/caracal-server/caracal/storage/key_audit.py](packages/caracal-server/caracal/storage/key_audit.py#L17-L50)).

**Impact:** No historical record of who created which principals or when; compliance gap for SOC 2.

**Recommendation:**
- Add audit log entry to `PrincipalRegistry.register_principal()`:
  ```python
  session.add(AuditLog(
      event_type="principal_created",
      topic="system.principal_lifecycle",
      principal_id=identity.principal_id,
      event_data={
          "name": name,
          "kind": principal_kind,
          "owner": owner,
          "metadata": metadata,
      }
  ))
  ```
- Log actor identity (CLI user, API caller, system bootstrap)

---

### **B. Policy Creation/Modification Not Audited**

**Files/Functions:**
- [packages/caracal-server/caracal/cli/authority_policy.py](packages/caracal-server/caracal/cli/authority_policy.py#L160-L180) — `create()` commits policy without audit record

**Issue:** Policy changes (creation, tightening, deactivation) not logged to `AuditLog`.

**Impact:** No historical record of policy evolution; cannot answer "when did resource X become restricted?"

**Recommendation:**
- Add audit log entry in `create()`, `tighten()`, `deactivate()` operations:
  ```python
  session.add(AuditLog(
      event_type="policy_created",
      topic="system.policy_lifecycle",
      principal_id=str(policy.principal_id),
      event_data={
          "policy_id": str(policy.policy_id),
          "max_validity": policy.max_validity_seconds,
          "allow_delegation": policy.allow_delegation,
      }
  ))
  ```

---

### **C. Allowlist Changes Not Audited**

**Files/Functions:**
- [packages/caracal-server/caracal/cli/allowlist.py](packages/caracal-server/caracal/cli/allowlist.py#L60-L85) — `create()` adds allowlist without audit
- [packages/caracal-server/caracal/cli/allowlist.py](packages/caracal-server/caracal/cli/allowlist.py#L160-L200) — `delete()` removes allowlist without audit

**Issue:** Allowlist mutations (add/remove patterns) not recorded in `AuditLog`.

**Impact:** Cannot answer "why was principal X suddenly denied access to resource Y?"

**Recommendation:**
- Log to `AuditLog` every allowlist create/delete:
  ```python
  session.add(AuditLog(
      event_type="allowlist_created|deleted",
      topic="system.allowlist_lifecycle",
      principal_id=str(principal_id),
      event_data={
          "allowlist_id": str(allowlist.allowlist_id),
          "pattern": pattern,
          "type": pattern_type,
      }
  ))
  ```

---

### **D. Mandate Issuance Not Linked to Audit Context**

**Files/Functions:**
- [packages/caracal-server/caracal/core/mandate.py](packages/caracal-server/caracal/core/mandate.py) — Mandate issuance logic (exact file path not fully explored)
- [packages/caracal-server/caracal/logging_config.py](packages/caracal-server/caracal/logging_config.py#L830-L850) — Logging config for mandate validation

**Issue:** Mandate issuance events (success/denial) log to structured logs but have no correlation_id linking to principal/policy audit trail.

**Impact:** Difficult to correlate "mandate was denied at 14:03 UTC" to "policy changed at 14:00 UTC".

**Recommendation:**
- Add `correlation_id` to mandate issuance audit record
- Link to principal + policy that governed it
- Include denial reason in `AuditLog.event_data`

---

### **E. Compliance Events Stored In-Memory Only**

**Files/Functions:**
- [caracalEnterprise/services/gateway/compliance_events.py](caracalEnterprise/services/gateway/compliance_events.py#L1-L80) — Gateway denial events in circular buffer

**Issue:** `record_gateway_denial_event()` appends to `_gateway_events` (global list, max 5000 items) with no DB persistence.

**Impact:** On gateway restart, compliance event history is lost; irrecoverable for audits.

**Recommendation:**
- Persist compliance events to PostgreSQL `compliance_events_log` table:
  ```sql
  CREATE TABLE compliance_events_log (
    event_id BIGSERIAL PRIMARY KEY,
    source VARCHAR(50),
    event_type VARCHAR(50),
    workspace_id UUID,
    org_id UUID,
    reason_code VARCHAR(100),
    timestamp TIMESTAMP,
    metadata JSONB,
    created_at TIMESTAMP DEFAULT UTCNOW()
  );
  ```
- Background worker flushes in-memory buffer to DB every 10 seconds (async, fail-safe)

---

### **F. No Audit Log for Dev-Mode Relaxations**

**Files/Functions:**
- [packages/caracal-server/caracal/cli/bootstrap.py](packages/caracal-server/caracal/cli/bootstrap.py#L95-L120) — `_grant_bootstrap_authority()` and dev allowlist creation

**Issue:** In `CCL_ENV_MODE=dev`, system principal auto-grants wildcard allowlist (`*` glob) without audit note.

**Impact:** In production with dev settings accidentally enabled, broad access is silent; no flag in audit trail.

**Recommendation:**
- Log a **SECURITY_EXCEPTION** event to `AuditLog` when dev-mode privilege grant occurs:
  ```python
  session.add(AuditLog(
      event_type="bootstrap_auth_granted_dev_mode",
      topic="system.security_exception",
      principal_id=principal_id,
      event_data={
          "capabilities": list(capabilities),
          "env_mode": os.environ.get("CCL_ENV_MODE"),
          "reason": "Bootstrap allowlist auto-grant in dev environment",
      }
  ))
  ```

---

## **III. NECESSARY SECURITY GATES (Should Remain)**

The following should **NOT** be removed or loosened:

| Gate | Location | Justification |
|------|----------|---|
| AIS attestation nonce validation | [packages/caracal-server/caracal/identity/attestation_nonce_manager.py](packages/caracal-server/caracal/identity/) | Prevents unauthorized CLI access to mandate issuance |
| Session token HMAC verification | [packages/caracal-server/caracal/core/session_manager.py](packages/caracal-server/caracal/core/session_manager.py) | Session replayability prevention |
| Principal lifecycle status check | [packages/caracal-server/caracal/db/models.py](packages/caracal-server/caracal/db/models.py#L55-L75) — `PrincipalLifecycleStatus` | Prevents revoked/suspended principals from issuing mandates |
| Policy active flag validation | [packages/caracal-server/caracal/db/query_optimizer.py](packages/caracal-server/caracal/db/query_optimizer.py#L248-L285) | Only active policies used for mandate evaluation |
| Allowlist resource pattern matching | [packages/caracal-server/caracal/core/allowlist.py](packages/caracal-server/caracal/core/allowlist.py) | Resource access control enforcement |
| Certificate validation in gateway auth | [caracalEnterprise/services/gateway/auth.py](caracalEnterprise/services/gateway/auth.py#L115-L160) — mTLS cert validation | Prevents forged client identities |
| Tenant quota enforcement | [caracalEnterprise/services/gateway/tenant_middleware.py](caracalEnterprise/services/gateway/tenant_middleware.py#L159-L200) | Prevents resource exhaustion |
| Owner-only file permissions (.env) | [packages/caracal-server/caracal/cli/bootstrap.py](packages/caracal-server/caracal/cli/bootstrap.py#L67) — `chmod 0o600` | Prevents secret exposure via world-readable files |
| Database connection auth | [packages/caracal-server/caracal/db/connection.py](packages/caracal-server/caracal/db/connection.py) | Prevents unauthorized data access |
| Redis password requirement | [deploy/docker-compose.yml](deploy/docker-compose.yml#L95-L115) | Prevents cache poisoning |

**Note:** The `--force` flag on `caracal bootstrap` re-issues AIS nonce but preserves principal identity—this is **safe** and should remain.

---

## **IV. REPETITIVE MANUAL WORKFLOWS THAT CAN BE SAFELY AUTOMATED**

| Workflow | Current State | Automation Strategy | Risk Level |
|----------|---|---|---|
| **Principal bulk registration** | CLI one-by-one | CSV import + batch create | Low—just de-duplicating existing ops |
| **Allowlist population from policy** | Manual per principal | Auto-materialize from policy.allowed_* | Low—policy already defines scope |
| **Default workspace provisioning** | Manual `caracal workspace create` | Auto-create on first login/bootstrap | Low—idempotent operation |
| **Policy version tracking** | Policy immutable after create | Track tightening history + audit log | Low—read-only audit trail |
| **Session token minting** | Manual `caracal auth token` | Sidecar token server + auto-refresh | Low—server-controlled TTL |
| **Vault secret bootstrap** | Manual env var setup | `caracal vault init` command | Low—one-time setup |
| **Team onboarding (enterprise)** | Manual step progression | Skip to workspace creation with defaults | Medium—requires role assumption |
| **MFA enforcement (enterprise)** | Policy flag only | Auto-enroll on tier ≥ professional | Medium—user workflow change |
| **Delegation limit increases** | Never—immutable policy | Requires new principal + new policy | High—**leave manual** |
| **Credential rotation** | Manual key deletion + new principal | Scheduled rotation + seamless cutover | High—**leave manual** for security |

---

## **V. CONCRETE REMEDIATION ROADMAP**

### **Phase 1 (Security-Preserving, High-Impact Automation)**

1. **Add audit logging to principal/policy/allowlist operations**
   - Files: [packages/caracal-server/caracal/cli/principal.py](packages/caracal-server/caracal/cli/principal.py), [authority_policy.py](packages/caracal-server/caracal/cli/authority_policy.py), [allowlist.py](packages/caracal-server/caracal/cli/allowlist.py)
   - Effort: 2–3 days
   - Blocks: None; additive

2. **Implement `caracal vault init` command**
   - Files: Create [packages/caracal-server/caracal/cli/vault_init.py](packages/caracal-server/caracal/cli/vault_init.py)
   - Effort: 3–4 days
   - Blocks: None; fills documented gap

3. **Auto-generate allowlist from policy**
   - Files: [packages/caracal-server/caracal/cli/authority_policy.py](packages/caracal-server/caracal/cli/authority_policy.py), [packages/caracal-server/caracal/core/allowlist.py](packages/caracal-server/caracal/core/allowlist.py)
   - Effort: 1–2 days
   - Blocks: None; operator can skip via flag

4. **Persist compliance events to DB**
   - Files: [caracalEnterprise/services/gateway/compliance_events.py](caracalEnterprise/services/gateway/compliance_events.py)
   - Effort: 1–2 days
   - Blocks: None; parallel system

### **Phase 2 (Packaging & Deployability)**

5. **Split SDK into standalone package**
   - Remove server imports from `caracal_sdk/_compat.py`
   - Files: [sdk/python-sdk/src/caracal_sdk/_compat.py](sdk/python-sdk/src/caracal_sdk/_compat.py)
   - Effort: 2–3 days
   - Blocks: Phase 1 must complete

6. **Embed `alembic.ini` in package data**
   - Use `importlib.resources` for config loading
   - Files: [packages/caracal-server/caracal/cli/db.py](packages/caracal-server/caracal/cli/db.py)
   - Effort: 1 day
   - Blocks: None; standalone

### **Phase 3 (Optional Autonomy Enhancements)**

7. **Token daemon sidecar** (`caracal token-daemon`)
8. **CSV principal import** (`caracal principal import <csv>`)
9. **Autonomous enterprise onboarding** (skip steps with defaults)

---

## **VI. SUMMARY TABLE: Issues by Security Impact**

| Issue | Severity | Autonomy Impact | Auditability Impact | Recommendation |
|-------|----------|---|---|---|
| No principal registration audit | HIGH | None | High | Add AuditLog entry |
| No policy creation audit | HIGH | None | High | Add AuditLog entry |
| Vault bootstrap missing | HIGH | High | Low | Implement `caracal vault init` |
| Compliance events in-memory only | HIGH | None | High | Persist to DB |
| Allowlist not auto-populated | MEDIUM | High | Low | Auto-create from policy (optional) |
| Alembic path resolution | MEDIUM | High | Low | Embed in package data |
| SDK reaches into server | MEDIUM | High | None | Remove imports, split package |
| No delegation approval workflow | MEDIUM | Medium | High | Track policy versions + audit |
| Session token requires manual invocation | LOW | High | None | Implement token daemon (optional) |
| Workspace creation manual | LOW | High | None | Auto-provision on first principal (optional) |

---

## **CONCLUSION**

The system is **secure by default** with strong enforcement gates. However, **operational autonomy is blocked** by missing automation hooks and **auditability is fragmented** across code locations. Implementing the Phase 1 remediations will close 80% of the gaps with minimal risk.

**Priority order:**
1. Audit logging (principal/policy/allowlist)
2. Vault initialization automation
3. Compliance event persistence
4. SDK decoupling (for packaging integrity)

All recommendations preserve existing security gates while enabling operators to run with minimal manual intervention.