# Semantic Clarity Audit: Caracal Ecosystem
**Date:** 2026-04-29  
**Scope:** caracal/ and caracalEnterprise/  
**Status:** Comprehensive read-only review complete

## Executive Summary

The Caracal ecosystem demonstrates strong architectural intent around "explicit authority" and "fail-closed" semantics. However, semantic clarity issues exist that create ambiguous contracts for humans and AI systems interpreting the code. Core problems:

1. **Overloaded terminology** - "decision", "validation", "denied" used across different domains with inconsistent semantics
2. **Implicit lifecycle rules** - Principal taxonomy rules not formally specified at runtime
3. **Unclear trust model** - Trust boundaries stated but enforcement patterns not explicit
4. **Ambiguous scope inheritance** - Caveat and delegation scope reduction rules lack formal contracts
5. **Runtime contract opacity** - When enforcement occurs vs when execution proceeds not made explicit

---

## FINDING 1: Overloaded "Decision" Terminology

### Evidence

Three distinct decision types with different semantics:

**1a) AuthorityDecision** [caracal/packages/caracal-server/caracal/core/authority.py:75-92]
```python
@dataclass
class AuthorityDecision:
    """
    Result of authority validation.
    
    Contains the decision outcome (allowed/denied) and the reason for the decision.
    """
    allowed: bool
    reason: str
    reason_code: Optional[str] = None
    boundary_stage: Optional[str] = None
    mandate_id: Optional[UUID] = None
    principal_id: Optional[UUID] = None
```
- **Semantics**: Pre-execution authorization boolean with reason chain
- **Failure mode**: No execution if `allowed=False`
- **Ledger event**: Mapped to `validated` or `denied` event type

**1b) LifecycleTransitionDecision** [caracal/packages/caracal-server/caracal/core/lifecycle.py:18-24]
```python
@dataclass(frozen=True)
class LifecycleTransitionDecision:
    """Decision metadata for a requested lifecycle transition."""
    
    allowed: bool
    reason: str
    principal_kind: str
    from_status: str
    to_status: str
```
- **Semantics**: State machine validation boolean for identity lifecycle
- **Failure mode**: Transition blocked; principal remains in source state
- **Ledger event**: Not directly logged; audit via principal state change

**1c) AllowlistDecision** [caracal/packages/caracal-server/caracal/core/allowlist.py:102-119]
```python
class AllowlistDecision:
    """
    Result of allowlist evaluation for agent resource restrictions.
    
    Attributes:
        allowed: Whether the resource is allowed
        reason: Human-readable explanation of the decision
    """
    allowed: bool
    reason: str
```
- **Semantics**: Pattern-matching result for resource filtering
- **Failure mode**: Resource execution denied at routing layer
- **Ledger event**: Not logged; enforcement is implicit

### Practical Impact

- **For humans**: Must know which decision type to check in different code contexts
  - `AuthorityDecision.allowed` checked before mandate execution [authority.py]
  - `LifecycleTransitionDecision.allowed` checked before state change [lifecycle.py]
  - `AllowlistDecision.allowed` checked before resource routing [allowlist.py]
- **For AI systems**: Type coercion failures if AI treats all "decision" objects identically
- **For testing**: Insufficient type differentiation may lead to false positive test coverage

### Root Cause

Reuse of `allowed: bool` pattern across three unrelated evaluation domains without discriminating type system.

### Supported Conclusion

**AMBIGUOUS CONTRACT**: The term "decision" is overloaded across authority, lifecycle, and allowlist domains. Each has different failure semantics and logging implications that are not explicit in the type system or naming.

---

## FINDING 2: "Validation" Terminology Ambiguity

### Evidence

**2a) Mandate validation** [caracal/packages/caracal-server/caracal/core/authority.py]
```
CANONICAL_AUTHORITY_VALIDATION_METHODS = ("validate_mandate",)
```
- Semantics: Pre-execution authorization check of mandate against policy/time/scope
- Method: `resolve_applicable_mandates_for_principal()` returns narrowed candidate list
- Runtime: Called at action boundary before delegation to provider

**2b) Caveat validation** [caracal/packages/caracal-server/caracal/core/caveat_chain.py:96-147]
```python
def verify_caveat_chain(*, hmac_key: str, chain: list[dict[str, Any]]) -> list[dict[str, Any]]:
    """Verify cumulative HMAC integrity for all caveat-chain entries."""
```
- Semantics: HMAC chain integrity check (cryptographic proof of append-only restrictions)
- Method: Compares computed digest to observed digest
- Runtime: Called before caveat chain evaluation

**2c) Validation count** [caracal/packages/caracal-server/caracal/redis/mandate_cache.py:280-301]
```python
def get_validation_count(self, mandate_id: UUID) -> int:
    """Get validation count for a mandate (for monitoring)."""
    count_key = f"{self.PREFIX_VALIDATION_COUNT}:{mandate_id}"
```
- Semantics: Telemetry counter; side effect of calling `validate_mandate()`
- Method: Redis counter increment
- Runtime: Monitoring only; not enforcement

**2d) Lifecycle validation** [caracal/packages/caracal-server/caracal/core/lifecycle.py:82-113]
```python
def validate_transition(
    self,
    *,
    principal_kind: str,
    from_status: str,
    to_status: str,
    attestation_status: str | None = None,
) -> LifecycleTransitionDecision:
    """Evaluate whether a lifecycle transition is allowed."""
```
- Semantics: State machine transition rule check
- Method: Lookup allowed transition table by principal kind
- Runtime: Called before `transition_lifecycle_status()`

### Practical Impact

- **For humans**: Unclear which validation check failed when debugging
  - Is mandate rejected because of scope mismatch? Time window? Caveat chain?
  - Reason codes exist but are not consistently applied across validators
- **For AI systems**: Cannot distinguish when to retry, when to escalate, when to fail safely
- **For auditing**: Validation events not unified under single ledger event type; no cross-check that all validations were performed before execution

### Root Cause

Five different validation concepts reuse the word "validate" without explicit naming to discriminate their contract:
1. Authorization check
2. Cryptographic integrity proof
3. Telemetry counter
4. State machine rule
5. Configuration format check

### Supported Conclusion

**IMPLICIT CONTRACTS**: "Validation" is used to mean five distinct operations with different failure modes and runtime semantics. Code must be read sequentially to understand which validation must pass before execution proceeds.

---

## FINDING 3: Unclear Principal Lifecycle Rules and Implicit Assumptions

### Evidence

**3a) Principal taxonomy** [caracal/packages/caracal-server/caracal/db/models.py:47-54]
```python
class PrincipalKind(str, Enum):
    """Behavioral principal taxonomy."""

    HUMAN = "human"
    SERVICE = "service"
    ORCHESTRATOR = "orchestrator"
    WORKER = "worker"
```

**3b) Lifecycle rules matrix** [caracal/packages/caracal-server/caracal/core/lifecycle.py:32-63]
```python
_DEFAULT_TRANSITIONS: dict[str, set[str]] = {
    _PENDING_ATTESTATION: {_ACTIVE, _EXPIRED, _REVOKED},
    _ACTIVE: {_SUSPENDED, _DEACTIVATED, _REVOKED},
    _SUSPENDED: {_ACTIVE, _DEACTIVATED, _REVOKED},
    _EXPIRED: {_ACTIVE, _DEACTIVATED, _REVOKED},
    _DEACTIVATED: {_ACTIVE, _REVOKED},
    _REVOKED: set(),
}

_NON_REACTIVATING_TRANSITIONS: dict[str, set[str]] = {
    _PENDING_ATTESTATION: {_ACTIVE, _EXPIRED, _REVOKED},
    _ACTIVE: {_DEACTIVATED, _REVOKED},
    _SUSPENDED: {_DEACTIVATED, _REVOKED},
    _EXPIRED: {_REVOKED},
    _DEACTIVATED: {_REVOKED},
    _REVOKED: set(),
}

_KIND_RULES: dict[str, dict[str, set[str]]] = {
    PrincipalKind.HUMAN.value: _DEFAULT_TRANSITIONS,
    PrincipalKind.SERVICE.value: _DEFAULT_TRANSITIONS,
    PrincipalKind.ORCHESTRATOR.value: _NON_REACTIVATING_TRANSITIONS,
    PrincipalKind.WORKER.value: _NON_REACTIVATING_TRANSITIONS,
}
```

**3c) Implicit semantics in code comments (found in tests, not code)**
[caracal/tests/unit/test_nested_workflows.py:14-15]
```python
class TestWorkerLifecycleStates:
    """Principal lifecycle state rules for worker and orchestrator kinds."""
```

### Problem Analysis

**Implicit assumption 1: Reactivation semantics**
- `_DEFAULT_TRANSITIONS` allows reactivation (DEACTIVATED → ACTIVE)
- `_NON_REACTIVATING_TRANSITIONS` forbids reactivation
- **Not explicit in code why**: No comment explains the distinction or its security/operational purpose
- **Runtime expectation**: Code assumes caller knows ORCHESTRATOR and WORKER are "non-reactivating"

**Implicit assumption 2: What "non-reactivating" means**
- For WORKER: Once deactivated, cannot return to ACTIVE (test confirms this [test_nested_workflows.py:20-24])
- For ORCHESTRATOR: Same rule as WORKER
- For HUMAN: Can reactivate after deactivation ([test_nested_workflows.py:26-45])
- For SERVICE: Can reactivate (uses _DEFAULT_TRANSITIONS)
- **Not explicit**: Why this distinction exists or what operational model it represents

**Implicit assumption 3: Attestation dependency**
[caracal/packages/caracal-server/caracal/core/lifecycle.py:119-127]
```python
if (
    kind in {PrincipalKind.ORCHESTRATOR.value, PrincipalKind.WORKER.value}
    and source == self._PENDING_ATTESTATION
    and target == self._ACTIVE
    and normalized_attestation != PrincipalAttestationStatus.ATTESTED.value
):
    return LifecycleTransitionDecision(
        allowed=False,
        reason=(
            f"{kind} principals can transition from pending_attestation to active "
            "only after attestation_status becomes 'attested'"
        ),
```
- Only ORCHESTRATOR and WORKER require attestation for activation
- HUMAN and SERVICE can activate without attestation
- **Not explicit in any configurable way**: The requirement is hard-coded in lookup location

### Practical Impact

- **For operators**: Cannot determine if a WORKER principal's non-reactivation is a security policy or implementation artifact
- **For AI systems**: Cannot reason about why a state transition was denied; no machine-readable contract explaining the rule
- **For testing**: Rules only validated through unit tests; no formal specification or configuration schema

### Root Cause

Lifecycle rules are implementation-defined through static lookup tables with implicit semantics encoded in naming (_DEFAULT vs _NON_REACTIVATING) and principal kind membership.

### Supported Conclusion

**IMPLICIT ARCHITECTURE**: Principal lifecycle rules are not explicitly defined as contracts. The distinction between HUMAN/SERVICE and ORCHESTRATOR/WORKER is implicit in code structure, not machine-readable, and not explicitly justified.

---

## FINDING 4: Ambiguous Scope and Delegation Inheritance

### Evidence

**4a) Scope types** [caracal/packages/caracal-server/caracal/core/mandate.py:163-164]
```python
def _validate_scope_subset(
    self,
    target_scope: List[str],
    source_scope: List[str]
) -> bool:
```

**4b) Pattern matching semantics** [caracal/packages/caracal-server/caracal/core/mandate.py:165-180]
```python
# Every item in target_scope must match at least one pattern in source_scope
for target_item in target_scope:
    match_found = False
    for source_item in source_scope:
        if self._match_pattern(target_item, source_item):
            match_found = True
            break
    
    if not match_found:
        return False

return True
```

**4c) Pattern matching implementation** [caracal/packages/caracal-server/caracal/core/mandate.py:182-192]
```python
def _match_pattern(self, value: str, pattern: str) -> bool:
    """
    Check if value matches pattern (supports wildcards).
    
    Args:
        value: The value to match
        pattern: The pattern to match against (supports * wildcard)
    
    Returns:
        True if value matches pattern, False otherwise
    """
    if value == pattern:
        return True
    if pattern.startswith("provider:") or value.startswith("provider:"):
        return False
    return fnmatchcase(value, pattern)
```

### Problem Analysis

**Implicit assumption 1: When is scope "safe"?**
- Pattern: `"provider:azure:resources/*/templates"` in source scope
- Request: `"provider:azure:resources/vms/templates"` in target scope
- **Current code behavior**: Returns `True` (wildcard matches)
- **Not explicit**: Whether this is "safe" scope reduction depends on provider semantics, not stated in contract
- **Risk**: Code assumes fnmatch glob patterns always represent "safe" restrictions

**Implicit assumption 2: Scope inheritance through delegation**
[caracal/examples/lynxCapital/INSTRUCTIONS.md:124-132]
```
2. **Regional Orchestrators** (5) - region-scoped authority.
3. **Invoice Intake Agents** (6 per region = 30) - read-only document scope.
...
8. **Audit Agents** (2 per region = 10) - record full delegation lineage.
```
- Documentation mentions "narrowly scoped" mandates
- **Not explicit in code**: How scopes are narrowed through delegation chain
- **Current implementation**: Each mandate independently defines action_scope and resource_scope
- **Missing contract**: What happens if delegation chain has inconsistent scopes?

**Implicit assumption 3: Caveat chain scope interaction**
[caracal/packages/caracal-server/caracal/core/caveat_chain.py:41-75]
- Caveats can restrict by action, resource, task, or expiry
- **Not explicit**: Interaction semantics when caveats and mandate scopes conflict
- Example undefined: 
  - Mandate allows `resource_scope: ["*"]`
  - Caveat restricts to `resource:prod/*`
  - Is prod/* enforced? Is mandate scope ignored?

### Practical Impact

- **For operators**: Cannot reason about whether delegated scopes are sufficiently restrictive without deep code reading
- **For AI systems**: No way to validate that scope inheritance follows principle of least privilege
- **For security review**: Scope restriction guarantees rest on glob matching semantics, not formal scope algebra

### Root Cause

Scope inheritance is implemented through pattern matching without explicit contract about when patterns are "safe" or how different scope restriction mechanisms (mandate scopes vs caveats vs allowlists) interact.

### Supported Conclusion

**UNCLEAR CONTRACTS**: Scope inheritance through delegation and caveat chains lacks explicit semantics. Pattern matching is used but no formal contract specifies what constitutes "safe" scope reduction.

---

## FINDING 5: Ambiguous Runtime Enforcement Point

### Evidence

**5a) Mandate validation called **before** execution**
[caracal/packages/caracal-server/caracal/core/authority.py:232-262]
```python
def resolve_applicable_mandates_for_principal(
    self,
    *,
    requested_action: str,
    requested_resource: str,
    caller_principal_id: Optional[str],
    current_time: Optional[datetime] = None,
    caveat_chain: Optional[list[dict[str, Any]]] = None,
    caveat_hmac_key: Optional[str] = None,
    caveat_task_id: Optional[str] = None,
) -> list[ExecutionMandate]:
    """Resolve active mandates applicable to a principal/action/resource.
    
    Resolution is internal and principal-driven: callers do not provide mandate IDs.
    This helper narrows candidates by subject binding, active time window, and
    action/resource scope match before runtime validation.
    """
```

**5b) Caveat evaluation timing unclear**
[caracal/packages/caracal-server/caracal/core/caveat_chain.py]
- `verify_caveat_chain()`: Verifies cryptographic integrity
- `evaluate_caveat_chain()`: Evaluates whether action/resource match caveat restrictions
- **Not explicit**: When evaluation happens relative to action execution

**5c) Allowlist enforcement point not documented**
[caracal/packages/caracal-server/caracal/core/allowlist.py:130-185]
- `AllowlistManager.check_resource_allowed()`: Returns decision but no context about when called
- **Missing**: Is this called at routing layer, execution layer, or both?

### Problem Analysis

**Implicit assumption 1: Pre-execution validation guarantees enforcement**
- Code comment says "before runtime validation" but no enforcing code shown
- Documentation (THREAT_MODEL.md line 119) states: "The first failure mode is enforcing policy too late, after side effects have already begun"
- **Not explicit in code**: What prevents a principal from executing an action despite negative validation result?

**Implicit assumption 2: Multiple enforcement layers (mandate, caveat, allowlist)**
- Three independent evaluation layers
- **Not explicit**: Interaction semantics if any layer denies
- **Uncertainty**: Does mandate denial prevent caveat evaluation? Are all three always checked?

**Implicit assumption 3: "Hard-cut" vs "soft-cut" enforcement**
[caracal/examples/lynxCapital/INSTRUCTIONS.md mentions hard-cut policy]
- **Not explicit in core code**: What makes enforcement "hard"?
- **THREAT_MODEL.md mentions** "fail-closed" but code doesn't explicitly implement fail-closed semantics

### Practical Impact

- **For operators**: Cannot verify that system enforces authorization before execution without code inspection
- **For compliance**: Cannot audit that enforcement actually occurred vs. was merely checked
- **For incident response**: Unsafe to assume enforcement happened; must verify at multiple layers

### Root Cause

Authority enforcement comment calls it "pre-execution" but actual enforcement mechanism (who calls validation, who checks result, what blocks execution) is not explicit in the code or runtime structure.

### Supported Conclusion

**UNSAFE WORDING**: Code uses pre-execution terminology but does not explicitly enforce that validation result blocks execution. Enforcement mechanism is implicit in calling layer, not explicit in the authority layer itself.

---

## FINDING 6: "Denied" vs "Revoked" Terminology Inconsistency

### Evidence

**6a) Event types mixing deny and revoke semantics**
[caracal/packages/caracal-server/caracal/db/models.py:748-752]
```python
event_type = Column(String(50), nullable=False, index=True)  # issued, validated, denied, revoked
```

**6b) Denied = authorization check failed**
[caracal/packages/caracal-server/caracal/core/authority.py:45]
```
MANDATE_MISSING = "AUTH_MANDATE_MISSING"
MANDATE_REVOKED = "AUTH_MANDATE_REVOKED"
...
ACTION_SCOPE_DENIED = "AUTH_ACTION_SCOPE_DENIED"
PREFIX = "AUTH_DENY"
```

**6c) Revoked = mandate lifecycle transition**
[caracal/packages/caracal-server/caracal/db/models.py:480-488]
```python
revoked = Column(Boolean, nullable=False, default=False, index=True)
revoked_at = Column(DateTime, nullable=True, index=True)
revoked_by = Column(
    PG_UUID(as_uuid=True),
    ForeignKey("principals.principal_id"),
    nullable=True,
)
```

### Problem Analysis

**Implicit assumption 1: Are denials immutable?**
- **Denied due to scope mismatch**: Reappear if scope is corrected
- **Revoked mandates**: Cannot be un-revoked; must be re-issued
- **Not explicit**: Contract difference between MANDATE_REVOKED (reason code) and revoked=True (state field)

**Implicit assumption 2: Denial vs revocation in lifecycle**
[caracal/packages/caracal-server/caracal/core/lifecycle.py]
```
PRINCIPAL_NOT_ACTIVE = "AUTH_PRINCIPAL_NOT_ACTIVE"
```
- When principal's lifecycle_status = SUSPENDED, actions are denied for that principal
- **Not explicit**: Are denials due to SUSPENDED equivalent to revocation? Should they be?

**Implicit assumption 3: Caveat vs mandate revocation**
- Mandates can be revoked (explicit state change)
- Caveats cannot be revoked (immutable chains)
- **Not explicit**: If a caveat becomes invalid, how is it handled? Is entire mandate treated as revoked?

### Practical Impact

- **For operators**: Unclear when to re-issue a mandate vs fix a condition
- **For auditing**: Cannot distinguish permanent revocation from temporary denial without reading event history
- **For AI systems**: Cannot reason about whether to retry after denial or give up

### Root Cause

Event logging uses both "denied" (for failed authorization checks) and "revoked" (for mandate state changes) without explicit contract distinguishing when each is appropriate.

### Supported Conclusion

**OVERLOADED SEMANTICS**: "Denied" and "revoked" are used to describe different failure modes (check failed vs state invalid) but the distinction is not explicit in the schema or validation logic.

---

## FINDING 7: Caveat Type Ambiguity and Normalization

### Evidence

**7a) Two caveat binding formats**
[caracal/packages/caracal-server/caracal/core/caveat_chain.py:48-66]
```python
if lowered.startswith("task-binding:"):
    binding_value = value.split(":", 1)[1].strip()
    if not binding_value:
        raise CaveatChainError("Task binding caveat must include a task identifier")
    return ParsedCaveat(CaveatType.TASK_BINDING, binding_value)

if lowered.startswith("task_binding:"):
    binding_value = value.split(":", 1)[1].strip()
    if not binding_value:
        raise CaveatChainError("Task binding caveat must include a task identifier")
    return ParsedCaveat(CaveatType.TASK_BINDING, binding_value)
```

**7b) Backward compatibility behavior**
[caracal/packages/caracal-server/caracal/core/caveat_chain.py:67-69]
```python
# Unprefixed caveats are treated as resource restrictions for backward compatibility.
return ParsedCaveat(CaveatType.RESOURCE, value)
```

### Problem Analysis

**Implicit assumption 1: Normalization of caveat format**
- Both `task-binding:` and `task_binding:` accepted
- **Not explicit**: Is this intentional backward compatibility or accidental dual support?
- **Risk**: New code might use one format; old code might produce other; they normalize to same type but origin is lost

**Implicit assumption 2: Unprefixed as resource caveat**
- Any unprefixed string is treated as resource restriction
- **Not explicit**: Contract defining when this default applies or should be avoided
- Example: User inputs `"foo"` intending an action restriction, gets resource restriction silently

**Implicit assumption 3: Caveat evaluation doesn't validate caveat format**
- parse_caveat() succeeds but evaluation might fail
- **Not explicit**: What happens if caveat value is invalid? (e.g., action: with no action name)

### Practical Impact

- **For operators**: Cannot predict caveat behavior from documentation; must read parser
- **For auditing**: Caveat "raw" field may not reconstruct to equivalent caveat due to normalization
- **For AI systems**: Cannot deterministically generate caveats from intent without knowing parser details

### Root Cause

Caveat parsing normalizes multiple input formats to single type without explicit contract about canonical format or equivalence rules.

### Supported Conclusion

**IMPLICIT NORMALIZATION**: Caveat formats are normalized (task-binding vs task_binding) and unprefixed strings default to resource restrictions. These normalization rules are not explicit in the contract.

---

## FINDING 8: Implicit Trust Model Between Caracal Core and Enterprise

### Evidence

**8a) Security boundaries stated but not enforced**
[caracalEnterprise/README.md:47-60]
```markdown
## Security Boundaries

Caracal Enterprise owns CCLE API routes, the gateway provider registry, 
gateway denial compliance events, and `gateway_vault` credentials. The OSS 
runtime owns broker execution and local vault adapters only. Do not add 
direct OSS imports of Enterprise modules, do not read `CCLE_*` settings 
from OSS runtime paths, and do not store Enterprise provider credentials 
outside the Enterprise gateway/vault path.
```

**8b) Configuration co-mingling**
[caracalEnterprise/services/api/src/caracal_api/config.py:29-60]
```python
"CCLE_VAULT_TOKEN": "CCL_VAULT_TOKEN",  # Maps Enterprise setting to Core setting
...
"CCL_VAULT_TOKEN": "enterprise-local-token",  # Override with Enterprise default
```

**8c) Session token semantics unclear**
[caracalEnterprise/src/contexts/WorkspaceStateContext.tsx:1-7]
```typescript
/*
 * Copyright (C) 2026 Garudex Labs.  All Rights Reserved.
 * Caracal, a product of Garudex Labs
 *
 * Single global store for canonical workspace state. Backend is the source of truth.
 */
```
- Documentation says "source of truth"
- **Not explicit**: What happens when frontend cache diverges from backend?

**8d) Provider routing without explicit contract**
[caracalEnterprise/README.md:62-66]
```markdown
Enterprise gateway traffic should use registry-backed provider routing:

- Send `X-Caracal-Provider-ID` for upstream selection.
- Do not use client-supplied `X-Caracal-Target-URL` in Enterprise gateway mode.
```
- **Not explicit**: How to detect that Enterprise mode is active?
- **Risk**: Missing header behavior not defined; fallback might be unsafe

### Practical Impact

- **For operators**: Unclear enforcement of security boundary between Core and Enterprise
- **For integrators**: No machine-readable contract about allowed/disallowed imports or setting reads
- **For security review**: Trust boundary relies on human code review, not technology enforcement

### Root Cause

Trust boundaries are documented as policies but not encoded in type system, namespacing, or runtime checks.

### Supported Conclusion

**DOCUMENTED BUT NOT ENFORCED**: Security boundaries between Caracal Core and Enterprise are clearly stated in documentation but not machine-enforced. Enforcement relies on developer discipline.

---

## FINDING 9: Permission Semantics vs Mandate Semantics Conflation

### Evidence

**9a) Two parallel permission models**
[caracalEnterprise/services/api/src/caracal_api/services/permissions.py:1-50]
```python
SENSITIVE_PAGE_IDS = {"api_keys", "secrets", "team", "billing", "archive"}
DEFAULT_ROLE_IDS = {"admin", "member", "viewer"}

DEFAULT_ROLES: Dict[str, Dict[str, Any]] = {
    "admin": {
        "permissions": _all_permissions(True, True, True),  # view, edit, lock
    },
    "member": {
        "permissions": _all_permissions(True, True, False),
    },
    "viewer": {
        "permissions": _all_permissions(True, False, False),
    },
}
```

**9b) Mandate model (core)**
[caracal/packages/caracal-server/caracal/db/models.py:436-520]
```python
class ExecutionMandate(Base):
    """Execution mandate for authority enforcement."""
    action_scope = Column(JSON, nullable=False)
    resource_scope = Column(JSON, nullable=False)
```

### Problem Analysis

**Implicit assumption 1: Permission model ≠ Mandate model**
- Enterprise uses role-based permissions (admin/member/viewer) for page access
- Core uses mandates (action/resource scope) for execution authority
- **Not explicit**: How they interact or whether they should be unified

**Implicit assumption 2: Dashboard permissions are separate enforcement**
[caracalEnterprise/services/api/tests/test_archive_lifecycle.py:52-53]
```python
def test_archive_page_is_permission_controlled_as_sensitive() -> None:
    source = _PERMISSIONS.read_text(encoding="utf-8")
```
- Archive page requires "admin" role
- **Not explicit**: Does archive operation also require mandate? Both? Either/or?

**Implicit assumption 3: Cascade between models**
- If user has "member" role but mandate allows only restricted action
- **Not explicit**: Which takes precedence? Can user who lacks permission override with mandate?

### Practical Impact

- **For operators**: Unclear whether to grant permissions or mandates for user access
- **For auditing**: Two consent paths (role permission + mandate) creates audit complexity
- **For AI systems**: Cannot determine if action was properly authorized without checking both layers

### Root Cause

Enterprise permission model and Core mandate model evolved separately with no explicit contract about their relationship.

### Supported Conclusion

**CONFLATED MODELS**: Enterprise permissions and Core mandates are separate but interact at the authorization boundary. Their interaction semantics are not explicit.

---

## FINDING 10: Implicit Behavior in Event Metadata

### Evidence

**10a) Untyped event metadata**
[caracal/packages/caracal-server/caracal/db/models.py:780-810]
```python
@property
def event_metadata(self) -> dict[str, Any]:
    metadata: dict[str, Any] = {}
    for entry in self.event_attributes:
        metadata[entry.attribute_key] = _decode_authority_attribute(
            entry.attribute_value,
            entry.value_type,
        )
    return metadata

@event_metadata.setter
def event_metadata(self, values: Optional[dict[str, Any]]) -> None:
    metadata = values or {}
    self.event_attributes = []
    for index, key in enumerate(sorted(metadata.keys())):
        encoded_value, value_type = _encode_authority_attribute(metadata[key])
        self.event_attributes.append(
            AuthorityEventAttribute(
                attribute_key=str(key),
                attribute_value=encoded_value,
                value_type=value_type,
                position=index,
            )
        )
```

**10b) Implicit encoding/decoding**
[caracal/packages/caracal-server/caracal/db/models.py:827-859]
```python
def _encode_authority_attribute(value: Any) -> tuple[str, str]:
    if value is None:
        return "", "null"
    if isinstance(value, bool):
        return ("true" if value else "false"), "bool"
    if isinstance(value, int):
        return str(value), "int"
    if isinstance(value, float):
        return str(value), "float"
    if isinstance(value, str):
        # ... continues
```

### Problem Analysis

**Implicit assumption 1: Metadata schema is uncontrolled**
- Any key-value pair can be added to event_metadata
- **Not explicit**: Contract specifying required vs optional metadata fields
- **Risk**: Events could lack critical context (which mandate, which policy evaluated, etc.)

**Implicit assumption 2: Type inference from value**
- Type is inferred by checking isinstance()
- **Not explicit**: What if a value is accidentally stored as wrong type? Silently misinterpreted on decode.

**Implicit assumption 3: Position ordering assumption**
```python
for index, key in enumerate(sorted(metadata.keys())):
```
- Metadata attributes are re-sorted by key on storage
- **Not explicit**: Is this for determinism or an implementation artifact? Does it matter for subsequent queries?

### Practical Impact

- **For auditing**: Cannot validate event completeness against schema
- **For integration**: External systems cannot validate metadata structure
- **For AI systems**: Cannot reason about missing required context in events

### Root Cause

Event metadata uses untyped dictionary with implicit encoding/decoding and undocumented schema.

### Supported Conclusion

**IMPLICIT STRUCTURE**: Event metadata lacks explicit schema contract. Type encoding/decoding is implicit and metadata completeness is unvalidated.

---

## SUMMARY TABLE: Issues by Severity & Category

| Finding | Category | Severity | Scope | Fixable |
|---------|----------|----------|-------|---------|
| 1. Overloaded "Decision" | Terminology | Medium | Type system | Yes - Create discriminating type hierarchy |
| 2. Ambiguous "Validation" | Terminology | High | Naming convention | Yes - Create validation-specific method names |
| 3. Implicit Lifecycle Rules | Architecture | High | Principal lifecycle | Yes - Formalize rules with config |
| 4. Unclear Scope Inheritance | Contract | High | Scope validation | Yes - Define formal scope algebra |
| 5. Implicit Enforcement Point | Runtime behavior | Critical | Authority layer | Yes - Explicit enforcement gate |
| 6. "Denied" vs "Revoked" | Terminology | Medium | Event logging | Yes - Distinguish event types |
| 7. Caveat Type Ambiguity | Contract | Medium | Parser | Yes - Single canonical format |
| 8. Implicit Trust Boundaries | Architecture | High | Core/Enterprise boundary | Yes - Runtime enforcement |
| 9. Permission/Mandate Conflation | Architecture | High | Auth model | Yes - Explicit precedence rule |
| 10. Implicit Event Metadata | Contract | Medium | Ledger events | Yes - Formalize schema |

---

## Recommended Actions (Prioritized)

### Priority 1: Critical Runtime Safety
- **Finding 5**: Explicitly encode enforcement gate in Authority layer's return type or wrapper
- **Finding 3**: Formalize principal lifecycle rules as machine-readable config with explicit justification

### Priority 2: High-Impact Clarity
- **Finding 2**: Rename validation methods with explicit operation names
- **Finding 4**: Document scope intersection/reduction semantics formally
- **Finding 8**: Add runtime checks for trust boundary violations

### Priority 3: Medium-Impact Maintainability
- **Finding 1**: Create decision interface hierarchy
- **Finding 6**: Separate denial event types from revocation event types
- **Finding 7**: Normalize caveat formats to single canonical form
- **Finding 9**: Explicit permission/mandate precedence rule
- **Finding 10**: Formalize event metadata schema
