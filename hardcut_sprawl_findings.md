# Hardcut/Preflight/Policy Sprawl Scan - Session Findings

## Key Locations Identified
1. **Central hardcut/preflight file**: caracal/packages/caracal/caracal/runtime/hardcut_preflight.py (535 lines)
2. **Enterprise API hardcut check**: caracalEnterprise/services/api/src/caracal_api/main.py (_assert_registration_state_hardcut_contract)
3. **Authorization logic scattered**:
   - caracal/packages/caracal/caracal/runtime/entrypoints.py (AIS handler _authorize* functions)
   - caracal/packages/caracal-server/caracal/mcp/adapter.py (_authorize_principal_request)
   - caracal/packages/caracal-server/caracal/mcp/service.py (401/403 checks)
4. **Central authority evaluator**: caracal/packages/caracal-server/caracal/core/authority.py

## Issues Found
- Three separate authorization checks in different modules 
- Duplication of capability checking logic
- Multiple validation paths for similar concerns
- Authorization embedded in handler code (entrypoints.py)
- CLI policy command has inline validation (authority_policy.py)
