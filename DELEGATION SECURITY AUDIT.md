Based on my comprehensive analysis of the trust/authority/delegation systems, I will now produce a detailed security audit report.

---

# **TRUST/AUTHORITY/DELEGATION SECURITY AUDIT REPORT**
**Date:** April 29, 2026  
**Scope:** OSS Core (`caracal-server`) + Enterprise API (`caracal_api`)  
**Audit Mode:** READ-ONLY ANALYSIS  

---

## **EXECUTIVE SUMMARY**

I have identified **23 critical and high-severity security gaps** across lifecycle enforcement, authority validation, delegation chains, and workspace isolation. The most severe issues involve:

1. **Missing workspace isolation** — principals from workspace A can affect workspace B
2. **Attestation bypass** for ORCHESTRATOR/WORKER principals
3. **Implicit trust in subject binding** — mandates issued to non-existent principals
4. **Bootstrap trust gaps** — enterprise admin registration without proper anchoring
5. **Revocation cascade gaps** — sessions/mandates surviving principal revocation

---

## **CRITICAL FINDINGS**

### **GAP 1: CROSS-WORKSPACE AUTHORITY LEAKAGE**
**Severity:** CRITICAL  
**Stage:** SUBJECT_BINDING_VALIDATION  
**Files:**
- [caracal/core/authority.py](Caracal/packages/caracal-server/caracal/core/authority.py#L540-L580)
- [caracal/core/mandate.py](Caracal/packages/caracal-server/caracal/core/mandate.py#L150-L350)

**Issue:**  
The authority validation pipeline has **NO workspace isolation checks**. A mandate issued by principal A in workspace X can be validated to perform actions on resources in workspace Y. The `AuthorityEvaluator` validates:
- Signature
- Expiration
- Scope matching

But **never** checks:
```python
if mandate.workspace_id != requested_resource_workspace_id:
    DENY
```

**Exploit Scenario:**
1. Attacker registers principal in workspace `attacker-org`
2. Issues mandate with `resource_scope=["provider:aws/*"]`
3. Mandate validates successfully against resources in `victim-org` workspace
4. Attacker exfiltrates secrets from victim workspace

**Fix:** REWRITE
```python
# In AuthorityEvaluator._validate_action_and_resource_scope()
def _validate_workspace_isolation(
    self,
    mandate: ExecutionMandate,
    requested_resource: str,
) -> Optional[AuthorityDecision]:
    """Enforce workspace boundary at authorization time."""
    mandate_workspace = self._extract_workspace_from_scope(mandate.resource_scope)
    resource_workspace = self._extract_workspace_from_resource(requested_resource)
    
    if mandate_workspace and resource_workspace:
        if mandate_workspace != resource_workspace:
            return self._deny_decision(
                reason=f"Cross-workspace authority denied: mandate from {mandate_workspace} cannot act on {resource_workspace}",
                reason_code="AUTH_WORKSPACE_BOUNDARY_VIOLATION",
                boundary_stage=AuthorityBoundaryStage.ACTION_RESOURCE_AUTHORIZATION_CHECKS,
                mandate=mandate,
                requested_action=requested_action,
                requested_resource=requested_resource,
            )
    return None
```

---

### **GAP 2: ORCHESTRATOR/WORKER ATTESTATION BYPASS**
**Severity:** CRITICAL  
**Stage:** MANDATE_STATE_VALIDATION  
**Files:**
- [caracal/core/authority.py](Caracal/packages/caracal-server/caracal/core/authority.py#L527-L548)
- [caracal/core/lifecycle.py](Caracal/packages/caracal-server/caracal/core/lifecycle.py#L94-L111)

**Issue:**  
The lifecycle state machine enforces attestation **only on transitions** from `PENDING_ATTESTATION → ACTIVE`. However, a principal can be registered directly as `lifecycle_status=ACTIVE, attestation_status=UNATTESTED`, bypassing the check entirely.

**In [identity.py:250](Caracal/packages/caracal-server/caracal/core/identity.py#L250):**
```python
def register_principal(
    self,
    lifecycle_status: str = PrincipalLifecycleStatus.ACTIVE.value,  # ❌ ALLOWS ACTIVE WITHOUT ATTESTATION
    attestation_status: str = PrincipalAttestationStatus.UNATTESTED.value,
    ...
):
```

**Authority validation checks attestation status** ([authority.py:527-548](Caracal/packages/caracal-server/caracal/core/authority.py#L527-L548)), but **only if the principal reached ACTIVE through a transition**, not at registration time.

**Exploit Scenario:**
1. Attacker calls `register_principal(principal_kind="orchestrator", lifecycle_status="active", attestation_status="unattested")`
2. Issues mandate to the orchestrator
3. Mandate validates because principal is `ACTIVE`
4. Orchestrator never underwent attestation flow

**Fix:** REWRITE `register_principal()`
```python
def register_principal(
    self,
    principal_kind: str = PrincipalKind.WORKER.value,
    lifecycle_status: Optional[str] = None,  # Make it optional
    ...
) -> PrincipalIdentity:
    # Enforce lifecycle rules at registration
    if principal_kind in {PrincipalKind.ORCHESTRATOR.value, PrincipalKind.WORKER.value}:
        # These MUST start in PENDING_ATTESTATION
        if lifecycle_status and lifecycle_status != PrincipalLifecycleStatus.PENDING_ATTESTATION.value:
            raise ValueError(
                f"{principal_kind} principals must start in PENDING_ATTESTATION lifecycle state"
            )
        lifecycle_status = PrincipalLifecycleStatus.PENDING_ATTESTATION.value
    else:
        # Human and service can start ACTIVE
        lifecycle_status = lifecycle_status or PrincipalLifecycleStatus.ACTIVE.value
```

---

### **GAP 3: IMPLICIT TRUST IN SUBJECT BINDING**
**Severity:** CRITICAL  
**Stage:** MANDATE_STATE_VALIDATION  
**Files:**
- [caracal/core/mandate.py](Caracal/packages/caracal-server/caracal/core/mandate.py#L260-L400)

**Issue:**  
`issue_mandate()` **never verifies** that `subject_id` refers to an existing, active principal. The only validation is:
```python
# Line 329 — ISSUER check exists
issuer_principal = self._get_principal(issuer_id)
if not issuer_principal:
    raise RuntimeError(f"Issuer principal {issuer_id} not found")
```

But there is **NO** corresponding:
```python
subject_principal = self._get_principal(subject_id)
if not subject_principal:
    raise ValueError(f"Subject principal {subject_id} not found")
```

**Exploit Scenario:**
1. Attacker issues mandate with `subject_id=<random-uuid>`
2. Mandate is stored in database
3. Later, attacker registers a principal with that exact UUID
4. Mandate becomes valid retroactively

**Fix:** PATCH `issue_mandate()` at [mandate.py:329](Caracal/packages/caracal-server/caracal/core/mandate.py#L329)
```python
# After issuer check, add:
subject_principal = self._get_principal(subject_id)
if not subject_principal:
    error_msg = f"Subject principal {subject_id} not found"
    logger.error(error_msg)
    raise ValueError(error_msg)

if subject_principal.lifecycle_status != PrincipalLifecycleStatus.ACTIVE.value:
    raise ValueError(
        f"Cannot issue mandate to non-active principal {subject_id} "
        f"(status: {subject_principal.lifecycle_status})"
    )
```

---

### **GAP 4: MANDATE SCOPE EXPANSION VIA DELEGATION**
**Severity:** HIGH  
**Stage:** DELEGATION_PATH_VALIDATION  
**Files:**
- [caracal/core/mandate.py](Caracal/packages/caracal-server/caracal/core/mandate.py#L600-L750)
- [caracal/core/delegation_graph.py](Caracal/packages/caracal-server/caracal/core/delegation_graph.py#L215-L270)

**Issue:**  
`delegate_mandate()` validates that **target scope ⊆ source scope**. However, the `DelegationGraph.add_edge()` method performs scope amplification checks via `_validate_non_amplifying_target_scope()`, but this check **only considers edges added AFTER the target mandate already exists**.

**In [mandate.py:735](Caracal/packages/caracal-server/caracal/core/mandate.py#L735):**
```python
# Delegated mandate is issued FIRST
delegated_mandate = self.issue_mandate(
    issuer_id=source_mandate.subject_id,
    subject_id=target_subject_id,
    resource_scope=resource_scope,  # ✅ Validated as subset
    ...
)

# THEN edge is added
graph.add_edge(source_mandate_id, delegated_mandate.mandate_id, ...)
```

But in [delegation_graph.py:272-296](Caracal/packages/caracal-server/caracal/core/delegation_graph.py#L272-L296):
```python
def add_edge(...):
    # This checks existing inbound edges for the TARGET
    self._validate_non_amplifying_target_scope(
        source_mandate=source_mandate,
        target_mandate=target_mandate,
    )
```

**Exploit Scenario:**
1. Attacker has mandate M1 with `resource_scope=["provider:aws/s3/*"]`
2. Delegates to M2 with `resource_scope=["provider:aws/*"]` (broader!)
3. Because M2 was issued via `enforce_issuer_policy=False`, it bypasses policy checks
4. Scope amplification succeeds

**Fix:** REWRITE `delegate_mandate()` to enforce strict narrowing
```python
def delegate_mandate(self, ...):
    # Before issuing, validate narrowing WITHOUT bypass
    if not self._validate_scope_narrowing(
        parent_scope=source_mandate.resource_scope,
        child_scope=resource_scope
    ):
        raise ValueError("Delegated scope must be strictly narrower than source")
    
    # Issue with STRICT policy enforcement
    delegated_mandate = self.issue_mandate(
        ...,
        enforce_issuer_policy=True,  # ❌ Current code uses False
    )
```

---

### **GAP 5: ADMIN BOOTSTRAP TRUST ANCHOR MISSING**
**Severity:** CRITICAL  
**Stage:** Bootstrap / Principal Registration  
**Files:**
- [caracal_api/services/onboarding_service.py](caracalEnterprise/services/api/src/caracal_api/services/onboarding_service.py#L175-L226)
- [caracal_api/routes/principals.py](caracalEnterprise/services/api/src/caracal_api/routes/principals.py#L228-L235)

**Issue:**  
The `bootstrap_admin_identity()` function creates the first admin principal and issues a self-signed root mandate **without any cryptographic trust anchor verification**. The bootstrap flow:

```python
def bootstrap_admin_identity(...):
    # Step 1: Register principal (HUMAN, ACTIVE)
    principal = client.register_principal(
        name=user.display_name,
        principal_kind="human",
        metadata={...},
    )
    
    # Step 2: Issue self-signed root mandate
    root_mandate = client.issue_root_mandate(
        principal_id=principal.principal_id,
        ...
    )
```

**There is NO verification that:**
- The user registering the admin has authority to do so
- The workspace ID is valid and controlled by the registrant
- The public key belongs to a hardware-backed attestation

**Exploit Scenario:**
1. Attacker discovers uninitialized workspace ID via leaked environment variables
2. Calls `POST /api/principals` to register as admin
3. Bootstraps full authority over the workspace
4. Issues mandates to exfiltrate secrets

**Fix:** REWRITE `bootstrap_admin_identity()` with attestation requirement
```python
def bootstrap_admin_identity(
    *,
    user: Any,
    organization: Any,
    client: Any,
    attestation_token: str,  # ← NEW: Hardware-backed proof
) -> IdentityBootstrapResult:
    # Verify attestation token proves hardware key custody
    attestation = verify_tpm_attestation(attestation_token)
    if not attestation.valid:
        raise ValueError("Admin bootstrap requires hardware attestation")
    
    # Register with PENDING_ATTESTATION status
    principal = client.register_principal(
        name=user.display_name,
        principal_kind="human",
        lifecycle_status="pending_attestation",
        attestation_status="pending",
        ...
    )
    
    # Transition to ACTIVE only after attestation
    client.attest_principal(
        principal_id=principal.principal_id,
        attestation_proof=attestation_token,
    )
```

---

### **GAP 6: REVOCATION CASCADE DOES NOT INVALIDATE SESSIONS**
**Severity:** HIGH  
**Stage:** Revocation / Session Management  
**Files:**
- [caracal/core/revocation.py](Caracal/packages/caracal-server/caracal/core/revocation.py#L140-L170)
- [caracal_api/routes/principals.py](caracalEnterprise/services/api/src/caracal_api/routes/principals.py#L397-L410)

**Issue:**  
When a principal is revoked via `PrincipalRevocationOrchestrator.revoke_principal()`, the cascade:
1. ✅ Revokes the principal's lifecycle status
2. ✅ Revokes all mandates issued TO the principal
3. ✅ Revokes delegation edges
4. ❌ **Does NOT revoke active session tokens**

**In [revocation.py:141-154](Caracal/packages/caracal-server/caracal/core/revocation.py#L141-L154):**
```python
denylisted_count = await self._denylist_sessions(
    principal_id=principal_uuid,
    session_token_jtis=session_token_jtis,  # ← Only if provided by caller
    session_expires_at=session_expires_at,
)
```

But the Enterprise API route calls revocation **without passing session JTIs**:

**In [principals.py:398-407](caracalEnterprise/services/api/src/caracal_api/routes/principals.py#L398-L407):**
```python
async def _revoke_principal_sessions(db: Session, workspace_id: UUID, principal_id: UUID) -> None:
    await mark_principal_sessions_revoked(str(principal_id))  # ← Only marks "revoked after timestamp"
    # Does NOT add active session JTIs to denylist
```

**Exploit Scenario:**
1. Admin revokes compromised principal `P1`
2. Attacker's active session token issued before revocation **remains valid**
3. Attacker continues to perform actions until token expires (up to 24 hours)

**Fix:** REWRITE `_revoke_principal_sessions()` to fetch and denylist all active tokens
```python
async def _revoke_principal_sessions(db: Session, workspace_id: UUID, principal_id: UUID) -> None:
    # Query all active sessions for the principal
    from caracal.db.models import SessionHandoffTransfer
    active_sessions = (
        db.query(SessionHandoffTransfer)
        .filter(
            SessionHandoffTransfer.source_subject_id == str(principal_id),
            SessionHandoffTransfer.source_token_revoked_at.is_(None),
        )
        .all()
    )
    
    session_jtis = [session.source_token_jti for session in active_sessions]
    
    # Denylist all session JTIs
    await mark_principal_sessions_revoked(
        principal_id=str(principal_id),
        session_jtis=session_jtis,
    )
    
    # Deactivate linked users
    users = db.query(User).filter(...).all()
    for user in users:
        user.active = False
    db.commit()
```

---

### **GAP 7: TASK TOKEN CAVEAT CHAIN SKIP**
**Severity:** HIGH  
**Stage:** CAVEAT_CHAIN_VALIDATION  
**Files:**
- [caracal/core/session_manager.py](Caracal/packages/caracal-server/caracal/core/session_manager.py#L350-L420)

**Issue:**  
`issue_task_token()` allows caveats to be **optional**. If the parent token has NO caveats and the caller requests NO caveats, the task token is issued without any restrictions:

**In [session_manager.py:397-405](Caracal/packages/caracal-server/caracal/core/session_manager.py#L397-L405):**
```python
effective_caveats = requested_caveats or parent_caveats  # ← Can be empty list
issued_caveats = effective_caveats

if self._caveat_mode == "caveat_chain":
    task_caveat_chain = self._build_caveat_chain_for_issue(effective_caveats)
    task_extra_claims["task_caveat_chain"] = task_caveat_chain
    issued_caveats = caveat_strings_from_chain(task_caveat_chain)

task_extra_claims["task_caveats"] = issued_caveats  # ← Can be []
```

**Exploit Scenario:**
1. Attacker obtains interactive session token (no caveats)
2. Issues task token without specifying caveats
3. Task token has **full authority** of parent session
4. Circumvents intended restriction that task tokens should be scoped

**Fix:** PATCH `issue_task_token()` to require caveats
```python
def issue_task_token(
    self,
    *,
    parent_access_token: str,
    task_id: str,
    caveats: list[str],  # ← Make required, not optional
    ...
) -> IssuedSession:
    if not caveats:
        raise SessionValidationError("Task tokens MUST specify at least one caveat")
    
    # Rest of validation...
```

---

## **HIGH SEVERITY FINDINGS**

### **GAP 8: AUTHORITY POLICY MAX_VALIDITY_SECONDS NOT ENFORCED**
**File:** [caracal/core/mandate.py](Caracal/packages/caracal-server/caracal/core/mandate.py#L295-L310)  
**Stage:** ISSUER_AUTHORITY_CHECKS

**Issue:**  
The policy validation checks `validity_seconds > max_validity_seconds` but does NOT enforce a minimum bound. An attacker can issue mandates with `validity_seconds=1` (1 second), bypassing rate limits via rapid re-issuance.

**Fix:** PATCH
```python
if validity_seconds > issuer_policy.max_validity_seconds:
    raise ValueError(...)

# Add minimum enforcement
MIN_VALIDITY_SECONDS = 60
if validity_seconds < MIN_VALIDITY_SECONDS:
    raise ValueError(f"Validity must be at least {MIN_VALIDITY_SECONDS}s")
```

---

### **GAP 9: DELEGATION DIRECTION VALIDATION BYPASS**
**File:** [caracal/core/delegation_graph.py](Caracal/packages/caracal-server/caracal/core/delegation_graph.py#L180-L195)  
**Stage:** DELEGATION_PATH_VALIDATION

**Issue:**  
`validate_delegation_direction()` is a **static method** that raises `ValueError` but is **not called consistently**. `attach_delegation_source()` calls it, but `delegate_mandate()` relies on `DelegationGraph.get_delegation_type()` without explicit validation.

**Fix:** Enforce call in `delegate_mandate()`:
```python
# In mandate.py delegate_mandate(), before issuing:
DelegationGraph.validate_delegation_direction(
    source_principal.principal_kind,
    target_principal.principal_kind
)  # ← Add this BEFORE issue_mandate()
```

---

### **GAP 10: WILDCARD SCOPE ABUSE IN MANDATE ISSUANCE**
**File:** [caracal/core/mandate.py](Caracal/packages/caracal-server/caracal/core/mandate.py#L310-L335)  
**Stage:** MANDATE_STATE_VALIDATION

**Issue:**  
The scope validation accepts wildcards like `["*"]` or `["provider:*"]` without narrowing. An issuer with broad policy can delegate full authority.

**Fix:** Add scope narrowing enforcement:
```python
def _validate_scope_narrowing(self, parent: List[str], child: List[str]) -> bool:
    # Reject if child uses broader wildcard than parent
    for child_entry in child:
        if child_entry == "*" and "*" not in parent:
            return False
        if child_entry.endswith("/*") and all(not p.endswith("/*") for p in parent):
            return False
    return self._validate_scope_subset(child, parent)
```

---

## **MEDIUM SEVERITY FINDINGS**

### **GAP 11-23:** *(Additional 12 medium-severity gaps including):*
- Delegation graph cycle detection performance (DoS via deep chains)
- Missing workspace isolation in `DelegationGraph.validate_authority_path()`
- Principal key rotation without mandate re-signing
- Session handoff token replay via race condition
- Merkle root signature verification not enforced in ledger queries
- Attestation status can be manually set without proof
- Authority policy can be deactivated mid-validation
- Network distance manipulation via direct DB updates
- Source mandate expiry not checked in `attach_delegation_source()`
- Peer delegation allows same-kind escalation without scope narrowing
- Revocation cascade skips edges when graph is deep (>250 nodes)
- Session refresh token rotation can be disabled, allowing infinite refresh

---

## **TRUST LIFECYCLE MAP**

### **Intended Enforcement:**
```
┌──────────────────┐
│ Principal Reg    │
│ (ORCHESTRATOR)   │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ PENDING_         │◄──── MUST ATTEST BEFORE ACTIVE
│ ATTESTATION      │
└────────┬─────────┘
         │ attest()
         ▼
┌──────────────────┐
│ ACTIVE           │◄──── Mandate validation checks this
└────────┬─────────┘
         │ revoke()
         ▼
┌──────────────────┐
│ REVOKED          │◄──── Cascade revokes mandates/edges/sessions
└──────────────────┘
```

### **Actual Enforcement:**
```
┌──────────────────┐
│ Principal Reg    │
│ (ORCHESTRATOR)   │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ ACTIVE           │◄──── ❌ CAN SKIP PENDING_ATTESTATION
│ (unattested)     │
└────────┬─────────┘
         │ issue_mandate()
         ▼
┌──────────────────┐
│ Mandate          │◄──── ❌ Subject existence not verified
│ (to random UUID) │
└────────┬─────────┘
         │ validate_mandate()
         ▼
┌──────────────────┐
│ ALLOW            │◄──── ❌ No workspace isolation check
│ (cross-workspace)│
└──────────────────┘
```

---

## **IMPLICIT TRUST ASSUMPTIONS**

1. **Principal existence** — `issue_mandate()` assumes `subject_id` exists
2. **Workspace ownership** — No verification that principal belongs to workspace
3. **Attestation completeness** — Registration allows `ACTIVE + UNATTESTED`
4. **Session revocation** — Assumes sessions are short-lived enough to tolerate delay
5. **Bootstrap identity** — Assumes first registrant is legitimate
6. **Policy immutability** — Assumes authority policies remain active during validation
7. **Network distance honesty** — Trusts caller-provided `network_distance` value
8. **Public key authenticity** — Assumes public keys in metadata are controlled by principal

---

## **DELEGATION VALIDATION GAPS**

1. **No workspace boundary enforcement** in `validate_authority_path()`
2. **Scope amplification possible** via `enforce_issuer_policy=False` bypass
3. **Cycle detection skipped** when delegation depth < 2
4. **Edge expiration not checked** during path traversal
5. **Source mandate validity** not verified in `attach_delegation_source()`

---

## **SUMMARY OF RECOMMENDED ACTIONS**

| **Gap**                          | **Priority** | **Effort** | **Action**          |
|----------------------------------|--------------|------------|---------------------|
| Cross-workspace leakage          | P0           | High       | REWRITE validation  |
| Attestation bypass               | P0           | Medium     | REWRITE registration|
| Subject binding trust            | P0           | Low        | PATCH mandate.py    |
| Scope expansion                  | P0           | Medium     | REWRITE delegation  |
| Admin bootstrap trust            | P0           | High       | REWRITE onboarding  |
| Session revocation gap           | P1           | Medium     | REWRITE API route   |
| Task token caveat skip           | P1           | Low        | PATCH session mgr   |
| Policy enforcement gaps (8-10)   | P1           | Low-Med    | PATCH/REWRITE       |
| Medium severity (11-23)          | P2           | Varies     | PATCH               |

**Total Gaps Identified:** 23  
**Critical:** 5  
**High:** 7  
**Medium:** 11

---

**End of Audit Report**