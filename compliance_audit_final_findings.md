# Final Compliance Audit: 8 Accepted Plan Items

## Audit Date: 2026-04-29
## Scope: Caracal Ecosystem (caracalEcosystem workspace)

### Summary
- **Total Items Audited**: 8
- **Fully Implemented**: 7
- **Drift Found**: 1 (item 5 - role permissions mismatch)

---

## Item 1: Unknown Workflow Actions/Conditions Must Fail Closed
**Status**: ✓ IMPLEMENTED  
**Location**: [caracalEnterprise/services/api/src/caracal_api/services/workflow_contract.py](caracalEnterprise/services/api/src/caracal_api/services/workflow_contract.py#L24-L30)
- `validate_workflow_action()` at lines 24-30 rejects unknown actions with error
- `validate_workflow_condition()` at lines 104-115 rejects unknown conditions with error
- Validation occurs at API ingestion via Pydantic validators in [WorkflowStepModel](caracalEnterprise/services/api/src/caracal_api/routes/workflows.py#L42-L59)

## Item 2: Workflow Action Parameters Must Be Validated Before Persistence/Execution
**Status**: ✓ IMPLEMENTED  
**Location**: [caracalEnterprise/services/api/src/caracal_api/services/workflow_contract.py](caracalEnterprise/services/api/src/caracal_api/services/workflow_contract.py#L60-L100)
- `validate_workflow_parameters()` validates all action-specific parameters
- Integrated into [WorkflowStepModel._validate_parameters()](caracalEnterprise/services/api/src/caracal_api/routes/workflows.py#L55-L57) via Pydantic `model_validator`
- Parameters validated AT API request time, before database persistence

## Item 3: Workflow Issuer Substitution Requires Explicit Executor Workflow Authority
**Status**: ✓ IMPLEMENTED  
**Location**: [caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py](caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py#L2324-L2329)
- Lines 2324-2329 check if `issuer_id != executor_id`
- If different, calls `_require_workflow_authority()` with action `workflow:act_as_issuer`
- **Test**: [test_workflow_security.py lines 71-84](caracalEnterprise/services/api/tests/test_workflow_security.py#L71-L84)

## Item 4: Delegate_Authority and Register_Principal Require Explicit Executor Workflow Authority
**Status**: ✓ IMPLEMENTED  
**Location**: [caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py](caracalEnterprise/services/api/src/caracal_api/integrations/caracal_core.py#L2377-L2382) and lines 2430-2434
- `delegate_authority` action (line 2377-2382): calls `_require_workflow_authority()` with action `workflow:create_authority_policy`
- `register_principal` action (line 2430-2434): calls `_require_workflow_authority()` with action `workflow:register_principal`
- **Tests**: [test_workflow_security.py lines 91-129](caracalEnterprise/services/api/tests/test_workflow_security.py#L91-L129)

## Item 5: Member Role Must Not Have Write:Workflows While Admin Retains It
**Status**: ⚠️ PARTIAL DRIFT FOUND
**Issue**: Permission system mismatch between API-level RBAC and UI page permissions

**API-Level (Correct)**:
- [rbac.py ROLE_PERMISSIONS](caracalEnterprise/services/api/src/caracal_api/middleware/rbac.py#L48-L76)
- Admin role: has `Permission.WRITE_WORKFLOWS` (line 60)
- Member role: does NOT have `Permission.WRITE_WORKFLOWS` (lines 64-74)
- API endpoints enforce this via `@Depends(require_permission("write:workflows"))`

**UI-Level (Drift)**:
- [permissions.py DEFAULT_ROLES](caracalEnterprise/services/api/src/caracal_api/services/permissions.py#L50-L76)
- Member role: `_all_permissions(True, True, False)` (line 58) grants edit:true to ALL pages including workflows
- This is UI-only permission and doesn't bypass API validation, but creates UI/API mismatch

**Expected**: Member role should have `edit:false` for workflows page in permissions.py
**Current**: Member role has `edit:true` for workflows in permissions.py (via _all_permissions)

## Item 6: OSS Mandate Revocation Allowed Only for Issuer, Subject, or Explicit Scoped Revoke_Mandate Authority
**Status**: ✓ IMPLEMENTED  
**Location**: [caracal/packages/caracal-server/caracal/core/mandate.py](caracal/packages/caracal-server/caracal/core/mandate.py#L585-L590)
- Lines 585-590 enforce revocation authority check
- Allows: revoker_id == mandate.issuer_id OR revoker_id == mandate.subject_id OR _has_revocation_authority()
- `_has_revocation_authority()` (lines 181-194) checks for active policy with `revoke_mandate` action on applicable resources
- **Tests**: [test_mandate_revocation_authority.py](caracal/tests/unit/core/test_mandate_revocation_authority.py)

## Item 7: Enterprise Gateway Revoke Path Must Not Bypass Revocation Authority
**Status**: ✓ IMPLEMENTED  
**Location**: [caracalEnterprise/services/api/src/caracal_api/routes/gateway.py](caracalEnterprise/services/api/src/caracal_api/routes/gateway.py#L1182-L1194)
- Lines 1182-1194 check authority BEFORE calling gateway/fallback
- Same authority check as OSS: issuer, subject, or _has_revocation_authority()
- Authority check occurs BEFORE attempting gateway proxy (line 1196-1206)
- **Tests**: [test_gateway_revocation_security.py](caracalEnterprise/services/api/tests/test_gateway_revocation_security.py)

## Item 8: Tests Should Cover Accepted Changed Behavior
**Status**: ✓ IMPLEMENTED

**Workflow security tests**:
- [test_workflow_security.py](caracalEnterprise/services/api/tests/test_workflow_security.py): 
  - Line 9-22: Unknown action/condition rejection
  - Line 23-36: Missing parameter validation
  - Line 37-47: Invalid parameter shape validation
  - Line 48-60: Unknown condition in evaluation
  - Line 71-84: Issuer substitution authority requirement
  - Line 91-129: Delegate_authority and register_principal authority requirements

**Gateway revocation security tests**:
- [test_gateway_revocation_security.py](caracalEnterprise/services/api/tests/test_gateway_revocation_security.py):
  - Line 16-40: Gateway rejects admin without revocation authority
  - Line 42-84: Gateway uses mandate manager for authorized fallback

**OSS mandate tests**:
- [test_mandate_revocation_authority.py](caracal/tests/unit/core/test_mandate_revocation_authority.py):
  - Lines 11-22: Authority check accepts scoped revoke policy
  - Lines 25-33: Authority check rejects policy without revoke action
  - Lines 36-57: Revocation accepts explicit authority
  - Lines 59-75: Revocation rejects missing authority
