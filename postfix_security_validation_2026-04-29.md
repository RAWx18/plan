# POST-FIX SECURITY VALIDATION
**Date:** 2026-04-29  
**Scope:** Uncommitted changes in /caracalEcosystem  
**Focus Areas:**
1. `/api/gateway/revoke` bypass fixes
2. Workflow parameter validation at boundaries  
3. Member role workflow write access
4. Core revocation authority enforcement

---

## VALIDATION RESULTS

### 1. `/api/gateway/revoke` BYPASS - FIXED ✓

**Location:** [caracalEnterprise/services/api/src/caracal_api/routes/gateway.py:1165-1245](caracalEnterprise/services/api/src/caracal_api/routes/gateway.py#L1165)

**Current Implementation:**
- Line 1178-1179: Validates user is admin: `current_user.get("is_admin") or current_user.get("role") == "admin"`
- Line 1200-1207: **NEW** - Enforces explicit revocation authority check BEFORE any revocation:
  ```python
  if (
      str(revoker_uuid) != str(row.issuer_id)
      and str(revoker_uuid) != str(row.subject_id)
      and not manager._has_revocation_authority(revoker_uuid, row)
  ):
      raise HTTPException(status_code=403, detail="Revoker lacks explicit revocation authority...")
  ```
- Line 1209-1220: Attempts gateway revocation (with proper auth headers)
- Line 1222-1232: Falls back to local mandate.revoke_mandate() with validated revoker

**Security Properties:**
- ✓ Authorization checked BEFORE gateway call
- ✓ Authorization checked BEFORE fallback local revocation
- ✓ Uses `manager._has_revocation_authority()` which enforces scoped policies
- ✓ Issuer/subject natural revocation retained

---

### 2. WORKFLOW PARAMETER VALIDATION AT BOUNDARIES - ENFORCED ✓

**Request Boundary Validation:**

Location: [caracalEnterprise/services/api/src/caracal_api/routes/workflows.py:25-87](caracalEnterprise/services/api/src/caracal_api/routes/workflows.py#L25)

- Pydantic field validators on `WorkflowStepModel`:
  - Line 54-56: `validate_workflow_action()` via @field_validator
  - Line 58-60: `validate_workflow_condition()` via @field_validator  
  - Line 62-64: `validate_workflow_parameters()` via @model_validator(mode="after")
- Applied to CreateWorkflowRequest and UpdateWorkflowRequest

**Execution Boundary Validation:**

Location: [caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py:2150-2152](caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py#L2150)

In `execute_workflow()`:
```python
action = validate_workflow_action(step.get("action"))
parameters = validate_workflow_parameters(action, step.get("parameters", {}))
condition = validate_workflow_condition(step.get("condition"))
```

In `_execute_workflow_action()` (line 2334-2335):
```python
action = validate_workflow_action(action)
parameters = validate_workflow_parameters(action, parameters)
```

**Validation Contract:** [caracalEnterprise/services/api/src/caracal_api/services/workflow_contract.py:1-140](caracalEnterprise/services/api/src/caracal_api/services/workflow_contract.py)
- `validate_workflow_action()`: Checks action in ALLOWED_WORKFLOW_ACTIONS
- `validate_workflow_parameters()`: Per-action validation:
  - issue_mandate: requires subject_id, resource_scope, action_scope, validity_seconds; validates UUID fields, string lists, positive ints
  - revoke_mandate: requires mandate_id; validates UUID format, optional string reason
  - delegate_authority: requires principal_id, max_validity_seconds, allowed_resource_patterns, allowed_actions
  - register_principal: requires name; validates string fields and metadata object
- `validate_workflow_condition()`: Validates ALLOWED_WORKFLOW_CONDITIONS with field requirements

**Security Properties:**
- ✓ Double validation (request + execution boundaries)
- ✓ Type-safe field validation (UUID, string lists, positive integers)
- ✓ Enum-based action/condition allowlisting
- ✓ Required field enforcement
- ✓ Fail-closed on validation failure (raises ValueError)

---

### 3. MEMBER ROLE LACKS WRITE WORKFLOWS - ENFORCED ✓

**Location:** [caracalEnterprise/services/api/src/caracal_api/middleware/rbac.py:50-90](caracalEnterprise/services/api/src/caracal_api/middleware/rbac.py#L50)

**Role Permissions Matrix:**

Role.ADMIN permissions (line 54-71):
- Includes: `Permission.WRITE_WORKFLOWS` ✓

Role.MEMBER permissions (line 72-82):
- List: READ_PRINCIPALS, READ_MANDATES, READ_POLICIES, READ_LEDGER, READ_ANALYTICS, READ_COMPLIANCE, READ_TEAM, WRITE_PRINCIPALS, WRITE_MANDATES, WRITE_POLICIES
- Notably: NO WRITE_WORKFLOWS ✓
- NOT included: MANAGE_TEAM, INVITE_MEMBERS, UPDATE_ROLES, REMOVE_MEMBERS, MANAGE_ORGANIZATION, MANAGE_SETTINGS, MANAGE_LICENSE, WRITE_WORKFLOWS

Role.VIEWER permissions (line 83-90):
- List: READ_PRINCIPALS, READ_MANDATES, READ_POLICIES, READ_LEDGER, READ_ANALYTICS, READ_COMPLIANCE
- No write permissions ✓

**Enforcement at Endpoints:**

[caracalEnterprise/services/api/src/caracal_api/routes/workflows.py](caracalEnterprise/services/api/src/caracal_api/routes/workflows.py):
- Line 119: `Depends(require_permission("write:workflows"))` on POST /workflows (create)
- Line 339: `Depends(require_permission("write:workflows"))` on PATCH /workflows/{id} (update)
- Line 429: `Depends(require_permission("write:workflows"))` on DELETE /workflows/{id}
- Line 490: `Depends(require_permission("write:workflows"))` on POST /workflows/{id}/execute

**Security Properties:**
- ✓ Member role explicitly does not include WRITE_WORKFLOWS
- ✓ All workflow write operations require permission
- ✓ Enforced via dependencies at endpoint layer
- ✓ Admin retains full write capability

---

### 4. CORE REVOCATION AUTHORITY ENFORCEMENT - IMPLEMENTED ✓

**New Method:** `_has_revocation_authority()` [caracal/packages/caracal-server/caracal/core/mandate.py:192-207](caracal/packages/caracal-server/caracal/core/mandate.py#L192)

```python
def _has_revocation_authority(self, revoker_id: UUID, mandate: ExecutionMandate) -> bool:
    policy = self._get_active_policy(revoker_id)
    if not policy:
        return False
    if not self._validate_scope_subset(["revoke_mandate"], policy.allowed_actions):
        return False

    revocation_resources = [
        f"mandate:{mandate.mandate_id}",
        f"principal:{mandate.issuer_id}",
        f"principal:{mandate.subject_id}",
    ]
    return any(
        self._validate_scope_subset([resource], policy.allowed_resource_patterns)
        for resource in revocation_resources
    )
```

**Authority Validation Rules:**
1. Get active policy for revoker_id (returns None if no policy = fail-closed)
2. Check policy has "revoke_mandate" in allowed_actions via _validate_scope_subset()
3. Check revoker has resource pattern matching EITHER:
   - mandate:{mandate_id} (scope to specific mandate)
   - principal:{issuer_id} (scope to mandate issuer)
   - principal:{subject_id} (scope to mandate subject)

**Integration in revoke_mandate()** [caracal/packages/caracal-server/caracal/core/mandate.py:576-590](caracal/packages/caracal-server/caracal/core/mandate.py#L576):

```python
if (
    revoker_id != mandate.issuer_id
    and revoker_id != mandate.subject_id
    and not self._has_revocation_authority(revoker_id, mandate)
):
    error_msg = (
        f"Revoker {revoker_id} does not have authority to revoke mandate {mandate_id}. "
        "Only the issuer, subject, or a principal with explicit revocation authority can revoke a mandate."
    )
    logger.warning(error_msg)
    raise ValueError(error_msg)
```

**Revocation Authority Hierarchy:**
1. Natural revocation: issuer OR subject (issuer_id == revoker_id OR subject_id == revoker_id)
2. Explicit revocation: active policy WITH "revoke_mandate" action AND matching resource pattern

**Tests:** [caracal/tests/unit/core/test_mandate_revocation_authority.py](caracal/tests/unit/core/test_mandate_revocation_authority.py)
- test_has_revocation_authority_accepts_scoped_revoke_policy: ✓ Scoped policy accepted
- test_has_revocation_authority_rejects_policy_without_revoke_action: ✓ Non-revoke policies rejected
- test_revoke_mandate_accepts_explicit_revocation_authority: ✓ Explicit authority permits revocation
- test_revoke_mandate_rejects_policy_without_revocation_authority: ✓ Rejects unauthorized revocation

**Security Properties:**
- ✓ Explicit revocation authority separate from general write permissions
- ✓ Scope subset validation ensures pattern matching enforced
- ✓ Fail-closed (no policy = no authority)
- ✓ Ledger event recorded for revocation with metadata
- ✓ Cache invalidation on revocation
- ✓ Cascade revocation honored via delegation_graph

---

## SUMMARY TABLE

| Check | Status | Evidence |
|-------|--------|----------|
| /api/gateway/revoke bypass fixed | ✓ PASS | Authorization check at lines 1200-1207 BEFORE any revocation |
| Workflow param validation at request boundary | ✓ PASS | Pydantic validators on WorkflowStepModel lines 54-64 |
| Workflow param validation at execution boundary | ✓ PASS | validate_workflow_* calls in execute_workflow() lines 2150-2152 |
| Member role lacks write:workflows | ✓ PASS | WRITE_WORKFLOWS absent from Role.MEMBER permissions (rbac.py:72-82) |
| Admin role retains write:workflows | ✓ PASS | WRITE_WORKFLOWS in Role.ADMIN permissions (rbac.py:66) |
| Core revocation authority enforced | ✓ PASS | _has_revocation_authority() method validates policy + resource patterns |
| Revocation authority in revoke_mandate() | ✓ PASS | Lines 576-590 call _has_revocation_authority() in logic |
| Gateway endpoint uses new authority check | ✓ PASS | Lines 1200-1207 call manager._has_revocation_authority() |
