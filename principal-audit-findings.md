# Principal Registration & Identity Audit - Findings

## Registration Flow
1. **PrincipalRegistry.register_principal** (identity.py:136)
   - Called via IdentityService wrapper (identity/service.py:23)
   - Enterprise API calls through caracal_core.py:2661
   - Checks for duplicate name GLOBALLY (not per-workspace)
   - Generates ES256 keypair in Vault via generate_and_store_principal_keypair
   - Creates PrincipalKeyCustody record with 1:1 relationship

2. **SpawnManager.spawn_principal** (spawn.py:70)
   - Idempotency via PrincipalWorkloadBinding with binding_type="spawn_idempotency"
   - workload field = f"{issuer_uuid}:{idempotency_key}"
   - Checks for duplicate name GLOBALLY before spawn (spawn.py:151)
   - Creates mandate with context_tags for bootstrap tracking

## Identity Establishment
- Principal.public_key_pem stored as PEM string in DB
- PrincipalKeyCustody: 1:1 per principal, unique=True on principal_id
- PrincipalKeyCustodyVault: Vault reference with vault_key_ref
- Keys generated ONLY in Vault (hard-cut requirement)
- attestation_status: UNATTESTED → PENDING (spawn) → ATTESTED (bootstrap)

## Uniqueness Constraints
1. **Principal.name**: UNIQUE globally (models.py:278)
2. **PrincipalKeyCustody.principal_id**: UNIQUE (models.py:364)
3. **Principal.principal_id**: PRIMARY KEY UUID

## Key Scoping (CRITICAL)
- Mandates scoped by issuer_id and subject_id (UUIDs)
- NOT scoped by principal name
- resolve_applicable_mandates_for_principal filters by subject_id == caller_uuid
- Caveat chains optional for narrowing

## Workload Bindings
- PrincipalWorkloadBinding.workload is NOT unique
- binding_type distinguishes spawn vs bootstrap vs custom
- Used for idempotent replay detection (workload = issuer_id:idempotency_key)

## Owner Field
- Principal.owner stored as String(255) with workspace_id
- NOT indexed as unique constraint
- Mandates checked for workspace isolation in enterprise API (caracal_core.py:2700)

## Bootstrap Flow (OSS)
- bootstrap.py:133 - looks for existing "system" principal
- Idempotent: skips creation if Principal.count() > 0 and not --force
- VULNERABLE: if "system" doesn't exist but other principals do, creates new system principal

## Enterprise Bootstrap
- register_principal in caracal_core.py requires workspace_id=owner
- No stable enterprise_user_metadata lookup visible in grep
- Direct name-based lookup in get_principal_by_name filters by Principal.owner
- get_any_principal_by_name for cross-tenant detection (used in sync)

## Task Binding
- PrincipalWorkloadBinding.workload is NOT a foreign key to tasks
- caveat_task_id passed to evaluate_caveat_chain but not enforcement-critical
- NO task-instance UUID in Principal or Mandate models
- Two parallel workers would need separate unique principal names
