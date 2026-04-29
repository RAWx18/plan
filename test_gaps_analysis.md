# Test Gap Analysis - Caracal Ecosystem

## Scope
Read-only scan of production code and test coverage across:
- `/caracal/packages/caracal-server/caracal/core/` (authority, lifecycle, delegation, etc.)
- `/caracalEnterprise/services/api/` (routes, integrations, middleware)
- Related test directories

## Key Production Areas Identified

### 1. Security Boundaries
- `caracal/core/authority.py` - AuthorityEvaluator
- `caracal_api/middleware/authority.py` - Authority checking
- `caracal/core/delegation_graph.py` - Delegation graph

### 2. Lifecycle & State Transitions
- `caracal/core/lifecycle.py` - PrincipalLifecycleStateMachine
- `caracal/db/models.py` - AuthorityPolicy, ExecutionMandate

### 3. Policy Validators
- `caracal_api/routes/policies.py` - Policy route handlers
- `caracal/core/mandate.py` - Policy enforcement in mandate issuance

### 4. Authentication & Authorization
- `caracal_api/middleware/authority.py` - Auth middleware
- `caracal/core/session_manager.py` - Session validation

### 5. Delegation
- `caracal/core/delegation_graph.py` - Graph-based delegation
- `caracal_api/routes/delegation.py` - Delegation routes

### 6. Unsafe Parsing
- `caracal/core/intent.py` - Intent parsing
- `caracal_api/routes/onboarding.py` - TAR archive parsing (lines 570-660)

## Existing Test Coverage

### Strong Coverage Areas
- Lifecycle state transitions (tests/unit/core/test_lifecycle.py)
- Basic authority validation (tests/unit/core/test_authority.py)
- Intent parsing validation (tests/unit/core/test_intent.py)
- Exception hierarchy (tests/unit/test_exceptions.py)

### Thin Coverage Areas
- Policy tightening/monotonic restrictions
- Delegation graph cycle detection
- Cross-workspace isolation
- TAR archive parsing edge cases
- Caveat chain complex scenarios

## Status
Analysis phase complete. Ready for detailed gaps report.
