Based on my read-only consistency review of the current changes across both repositories, I can now provide the comprehensive verdict:

---

## CONSISTENCY REVIEW - FINAL VERDICT

### BLOCKERS

#### 1. **Workflow Contract Validation - Incomplete Implementation**
**Path**: [caracalEnterprise/services/api/src/caracal_api/services/workflow_contract.py](caracalEnterprise/services/api/src/caracal_api/services/workflow_contract.py)  
**Severity**: HIGH - Breaks fail-closed guarantee on workflow creation

**Issue**: The `validate_workflow_parameters()` function is defined but **NOT called during workflow creation** in [create_workflow](caracalEnterprise/services/api/src/caracal_api/routes/workflows.py#L122). Validation only occurs at runtime in `_execute_workflow_action()`.

**Evidence**:
- [workflows.py:L148-150](caracalEnterprise/services/api/src/caracal_api/routes/workflows.py#L148-L150): Steps converted via `step.model_dump()` and stored directly to DB as JSONB
- [workflows.py:L40-51](caracalEnterprise/services/api/src/caracal_api/routes/workflows.py#L40-L51): Pydantic validators validate individual steps at request time but are NOT re-validated at storage
- [caracal_core.py:L2335-2336](caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py#L2335-L2336): Validators called in `_execute_workflow_action()`, not at create time

**Risk**: Malformed workflow JSON can be persisted and fail at execution time rather than creation time, breaking the fail-closed contract. Stored steps bypass contract validation for later runtime.

**Requirement**: Workflow steps must be validated at creation time, not execution time.

---

#### 2. **Enterprise Workflow Authority Not Integrated with OSS Mandate Revocation**
**Path**: [caracal/packages/caracal-server/caracal/core/mandate.py:L191-205](caracal/packages/caracal-server/caracal/core/mandate.py#L191-L205)  
**Severity**: HIGH - Integration gap between OSS and Enterprise

**Issue**: Enterprise workflow execution via `_require_workflow_authority()` is not linked to the new `_has_revocation_authority()` method in OSS MandateManager.

**Evidence**:
- [mandate.py:L196](caracal/packages/caracal-server/caracal/core/mandate.py#L196): `_has_revocation_authority()` validates `"revoke_mandate" in policy.allowed_actions`
- [caracal_core.py:L2391-2395](caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py#L2391-L2395): Enterprise `revoke_mandate` action does NOT check `_require_workflow_authority()` before calling `self.revoke_mandate()`
- Result: An Enterprise workflow executor with no explicit revocation authority can revoke mandates via workflow if OSS MandateManager only checks issuer/subject

**Risk**: Privilege escalation: workflow execution bypasses the new resource-pattern matching revocation authority.

**Requirement**: Enterprise `_execute_workflow_action("revoke_mandate")` must call `_require_workflow_authority()` before revocation.

---

#### 3. **RBAC Permission Mismatch - WRITE_WORKFLOWS Removed But Still Required**
**Path**: [caracalEnterprise/services/api/src/caracal_api/routes/workflows.py:L119](caracalEnterprise/services/api/src/caracal_api/routes/workflows.py#L119)  
**Severity**: HIGH - Breaks admin workflow creation

**Issue**: The diff shows `Permission.WRITE_WORKFLOWS` removed from ADMIN role in rbac.py, but [workflows.py:L119](caracalEnterprise/services/api/src/caracal_api/routes/workflows.py#L119) still requires `depend(require_permission("write:workflows"))` for create_workflow.

**Evidence**:
- Diff line: `- Permission.WRITE_WORKFLOWS,` removed from ADMIN role
- Current: [rbac.py:L66](caracalEnterprise/services/api/src/caracal_api/middleware/rbac.py#L66) shows `Permission.WRITE_WORKFLOWS,` in ADMIN list (permission still defined but may be removed from role mapping)
- [routes/workflows.py:L119](caracalEnterprise/services/api/src/caracal_api/routes/workflows.py#L119): Endpoint requires `require_permission("write:workflows")`

**Risk**: If permission is removed from ADMIN role, admins receive 403 Forbidden on workflow creation even though they should have workflow authority via `_require_workflow_authority()` checks during execution.

**Requirement**: Either (a) keep WRITE_WORKFLOWS in ADMIN role, or (b) remove the permission check from create_workflow and rely solely on `_require_workflow_authority()` at execution.

---

### NON-BLOCKERS

#### 1. **Duplicate Field Definition - Minor Code Quality Issue**
**Path**: [caracalEnterprise/services/api/src/caracal_api/routes/workflows.py:L86-87](caracalEnterprise/services/api/src/caracal_api/routes/workflows.py#L86-L87)  
**Severity**: LOW

**Issue**: `workspace_id` defined twice in `WorkflowResponse` model (lines 86-87). Pydantic silently uses the second definition.

**Impact**: No functional impact, but reduces code clarity and may confuse future maintainers.

---

#### 2. **OSS Mandate Revocation Authority - Coherent Semantics**  
**Path**: [caracal/packages/caracal-server/caracal/core/mandate.py:L579-590](caracal/packages/caracal-server/caracal/core/mandate.py#L579-L590)  
**Severity**: NONE - Working as designed

**Finding**: The new `_has_revocation_authority()` check properly enforces:
- Issuer/subject (existing logic) OR
- Admin with "revoke_mandate" action AND matching resource pattern (new stricter logic)

**Evidence**:
- [mandate.py:L203-205](caracal/packages/caracal-server/caracal/core/mandate.py#L203-L205): Resource patterns must match `mandate:{id}`, `principal:{issuer}`, or `principal:{subject}` 
- Fail-closed: If no patterns match, raises ValueError

**Verdict**: Mandate revocation semantics remain coherent and more restrictive (improved security). Existing admins may need policy updates to include resource patterns, but this is correct behavior for stricter RBAC.

---

#### 3. **Workflow Contract Helpers - Validation Reuse Correct**
**Path**: [caracalEnterprise/services/api/src/caracal_api/services/workflow_contract.py](caracalEnterprise/services/api/src/caracal_api/services/workflow_contract.py)  
**Severity**: NONE - Working as designed

**Finding**: Route-time validation (Pydantic) and runtime validation (caracal_core) correctly use the same `validate_workflow_action()`, `validate_workflow_condition()`, and `validate_workflow_parameters()` helpers.

**Evidence**:
- [workflows.py:L40-51](caracalEnterprise/services/api/src/caracal_api/routes/workflows.py#L40-L51): Pydantic validators import and call the workflow_contract helpers
- [caracal_core.py:L2335-2336](caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py#L2335-L2336): Runtime execution re-validates with same helpers
- Defense-in-depth: Validation occurs twice (creation + execution)

**Verdict**: Helpers are properly centralized and reused. No duplication detected.

---

#### 4. **Stored Workflow JSON - Fail-Closed Execution Path**
**Path**: [caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py:L2512](caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py#L2512)  
**Severity**: NONE - Working as designed

**Finding**: `_execute_workflow_action()` includes final `else` clause that raises `ValueError("Unsupported workflow action: {action}")` for unknown actions. Stored JSON with unrecognized actions will fail execution, not silently succeed.

**Evidence**:
- [caracal_core.py:L2510-2512](caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py#L2510-L2512): Explicit `else` clause with ValueError
- All 4 actions (issue_mandate, revoke_mandate, delegate_authority, register_principal) explicitly handled
- Unknown condition types now fail properly due to Pydantic validators

**Verdict**: Fail-closed semantics enforced at runtime execution.

---

#### 5. **Test Coverage - Adequate for Changed Behavior**
**Path**: [caracalEnterprise/services/api/tests/test_workflow_security.py](caracalEnterprise/services/api/tests/test_workflow_security.py)  
**Severity**: NONE - Sufficient coverage

**Finding**: Tests cover:
- Unknown action rejection (line 28)
- Unknown condition rejection (line 18)
- Missing parameter rejection (line 37)
- Invalid parameter shape rejection (line 50)
- Authority enforcement for issuer substitution (line 75)
- Authority enforcement for delegation (line 99)

**Evidence**:
- [test_workflow_security.py:L18-60](caracalEnterprise/services/api/tests/test_workflow_security.py#L18-L60): All contract validation paths tested
- [test_workflow_security.py:L65-128](caracalEnterprise/services/api/tests/test_workflow_security.py#L65-L128): Authority integration tested
- Pydantic ValidationError properly propagated

**Verdict**: Test coverage adequate for validation and authority changes.

---

## ACCEPTANCE VERDICT

**❌ REJECTION** - Changes cannot be merged pending resolution of BLOCKERS.

### Required Fixes Before Acceptance:

1. **Add validation at workflow creation time** - Call `validate_workflow_parameters()` in [create_workflow](caracalEnterprise/services/api/src/caracal_api/routes/workflows.py#L122) before DB persistence
   
2. **Add revocation authority check to workflow action** - Insert `_require_workflow_authority()` in [caracal_core.py:L2389](caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py#L2389) before `revoke_mandate()` call

3. **Reconcile RBAC permission mapping** - Either retain `Permission.WRITE_WORKFLOWS` in ADMIN role OR remove `require_permission("write:workflows")` from workflow endpoints and rely entirely on `_require_workflow_authority()` validation logic

All three blockers must be resolved for consistent contract validation across OSS and Enterprise layers.