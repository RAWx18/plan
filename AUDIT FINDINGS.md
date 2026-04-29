Based on my thorough read-only audit of both `/home/raw/Documents/workspace/caracalEcosystem/caracal` and `/caracalEnterprise`, I have identified the ledger and state integrity infrastructure. Here are my findings:

## AUDIT FINDINGS: Ledger & State Integrity

### 1. LEDGER & AUDIT LOG ARCHITECTURE ✓

**Immutable Event Storage — PostgreSQL-backed, append-only:**

- [AuthorityLedgerEvent](caracal/packages/caracal-server/caracal/db/models.py#L1): Authority decisions (issued, validated, denied, revoked)
  - `event_type` enum: "issued", "validated", "denied", "revoked"
  - `principal_id`, `mandate_id`, `decision`, `denial_reason`, `requested_action`, `requested_resource`
  - Auto-incrementing `event_id` (BigInteger PK)
  - Immutable: only written, never updated/deleted
  - Composite index: `(principal_id, timestamp)` for range queries

- [LedgerEvent](caracal/packages/caracal-server/caracal/db/models.py#L95): Resource usage tracking
  - Auto-increment `event_id` (BigInteger)
  - Quantity as Decimal(20,6) for precision
  - Merkle root FK for cryptographic integrity
  - Never modified post-insertion

- [AdminAuditLog](caracalEnterprise/services/api/src/caracal_api/models/audit_log.py#L1): Enterprise admin actions
  - Coupon creation, invoice adjustment, credit application, pricing overrides
  - Nullable `admin_user_id` (for super-admin OAuth actions)
  - Full context in JSONB `details` column
  - IP address tracking
  - Created timestamp (immutable)
  - Indexes on: user, action, (resource_type, resource_id), created_at

- [MeteringEvent](caracalEnterprise) with `idempotency_key` column: Prevents duplicate ingestion
  - HMAC signature validation
  - Unique constraint: `(workspace_id, idempotency_key)` with partial index
  - Sequence number per organization

---

### 2. TASK/MANDATE STORAGE ✓

**Execution Mandate State — Durable, queryable:**

- [ExecutionMandate](caracal/packages/caracal-server; referenced throughout)
  - `mandate_id` (UUID PK)
  - `subject_id`, `issuer_id` (delegation relationships)
  - `revoked` flag with `revoked_at` timestamp
  - `resource_scope`, `action_scope` (parsed into relational tables in v0.3)
  - `valid_from`, `valid_until` (temporal validity)

- [SessionHandoffTransfer](caracal/packages/caracal-server/caracal/db/migrations/versions/q6r7s8t9u0v1_session_handoff_transactional_state_hardcut.py#L1): One-time token narrowing
  - `handoff_jti` (unique), `source_token_jti`, `target_subject_id`
  - `transferred_caveats`, `source_remaining_caveats` (attenuation tracking)
  - `issued_at`, `source_token_revoked_at`, `consumed_at` (lifecycle)
  - Composite indexes for revocation checks

---

### 3. SYNCHRONIZATION & TRANSACTION SAFETY ✓

**Atomic Session Scope Management:**

- [DatabaseConnectionManager.session_scope()](caracal/packages/caracal-server/caracal/db/connection.py#L200)
  - Context manager: auto-commit on success, rollback on exception
  - Transactional scope ensures consistency
  - `session.flush()` for ID assignment without commit
  - Error logging on rollback

**Handoff Transfer Persistence:**
- [SessionManager._record_handoff_transfer()](caracal/packages/caracal-server/caracal/core/session_manager.py#L900)
  - Single DB transaction: inserts transfer row + sets `source_token_revoked_at` atomically
  - **Critical finding**: If DB insert fails, source scope is **NOT** narrowed (source token remains valid)
  - Denylist marked after successful DB commit only

**Revocation Cascade — Leaves-first ordering:**
- [PrincipalRevocationOrchestrator._resolve_leaves_first_order()](caracal/packages/caracal-server/caracal/core/revocation.py#L290)
  - DFS traversal: deepest nodes revoked first
  - Prevents orphaned delegation chains
  - Async threshold: switches to background jobs for >250 principals

---

### 4. REDACTION & SENSITIVE DATA HANDLING ✓

**Structured Logging with Redaction:**
- [_redact_sensitive_values()](caracal/packages/caracal-server/caracal/logging_config.py#L60)
  - Recursive redaction of keys matching: `password`, `token`, `secret`, `key`, `cipher`, `credential`
  - Applied before log rendering
  - Tests confirm: `password`, `access_token`, `api_secret` are redacted to `"[REDACTED]"`

**API Key Masking:**
- [_credential_payload()](caracalEnterprise/services/api/src/caracal_api/routes/ccle.py#L99)
  - Response payload: `ccle_api_key: None` (never exposed)
  - Safe mask shown: `ccle_****` prefix with dots
  - Raw key returned **only** on rotation endpoint, never in read endpoints

**Vault-backed Secret Storage:**
- [CaracalVault._store_key_in_vault()](caracal/packages/caracal-server/caracal/core/vault.py#L800)
  - Secrets stored in managed vault, not in database
  - Reveal endpoint with 30-second TTL
  - Admin audit events on reveal/rotate/revoke

**Evidence — CCLE credentials test confirms non-disclosure:**
> `test_credential_payload_hides_plaintext_key_for_non_admin()` and `test_credential_payload_hides_plaintext_key_for_admin()` both verify `ccle_api_key: None` in response

---

### 5. IDEMPOTENCY & REPLAY SAFETY ✓

**HMAC-Signed Metering Events:**
- [MeteringEngine.compute_hmac()](caracalEnterprise/services/api/src/caracal_api/services/metering_engine.py#L120)
  - SHA256(workspace_id:sequence:event_type:timestamp:metadata_hash)
  - [MeteringEngine.verify_hmac()](caracalEnterprise/services/api/src/caracal_api/services/metering_engine.py#L135) uses constant-time comparison

**Idempotency Keys:**
- [MeteringEngine.compute_idempotency_key()](caracalEnterprise/services/api/src/caracal_api/services/metering_engine.py#L116)
  - SHA256(workspace_id:sequence:event_type:timestamp_iso:metadata_hash)
  - Unique constraint on `(workspace_id, idempotency_key)` with partial index
  - [ingest_signed_event()](caracalEnterprise/services/api/src/caracal_api/services/metering_engine.py#L240) checks exists before insert
  - Returns `(event, was_created)` tuple; duplicates counted separately

**Spawn Operation Idempotency:**
- [PrincipalSpawner._find_existing_spawn()](caracal/packages/caracal-server/caracal/core/spawn.py#L430)
  - Stores binding marker: `{issuer_id}:{idempotency_key}` in `PrincipalWorkloadBinding`
  - Retrieves via tag: `spawn:idempotency:{idempotency_key}`
  - Returns cached `SpawnResult` if already executed

**Nonce-based Replay Protection:**
- [ReplayProtection.check_nonce()](caracalEnterprise/services/gateway/replay_protection.py#L70)
  - Redis SETNX (atomic set if not exists) with TTL
  - Tracks used nonces with 300-second window
  - Return value: `True` = new, `False` = replay detected
  - Fails closed on Redis errors (denies request)

**Correlation IDs:**
- [add_correlation_id()](caracal/packages/caracal-server/caracal/logging_config.py#L50)
  - Context variable: stores correlation ID in thread-local storage
  - Automatically appended to structured logs
  - AuditLog table has `correlation_id` column with index

---

### 6. EVENT COMPACTION & RETENTION ✓

**Archive Records — Soft Delete Pattern:**
- [ArchiveRecord](caracalEnterprise/services/api/src/caracal_api/models/archive.py#L1)
  - `snapshot` (JSONB): stores full state before deletion
  - `archived_at`, `archived_by` (creation metadata)
  - `retention_days` (configurable TTL, default unset = manual)
  - `auto_delete_at` (calculated from retention_days)
  - `purged_at`, `purged_reason`: final deletion proof
  - `archive_ledger_event_id`, `purge_ledger_event_id`: cryptographic proof chain
  - Composite indexes: (workspace_id, archive_status), (workspace_id, resource_type), auto_delete_at

**Purge Automation:**
- [purge_due_archive_records()](caracalEnterprise/services/api/src/caracal_api/services/archive.py#L120)
  - Finds records with `archive_status='archived'` and `auto_delete_at <= now()`
  - Sets `purge_due=True`, calls purge_archive_record()
  - Records proof: `purge_ledger_event_id` references authority ledger

**Audit Log Archival:**
- [AuditManager.archive_old_logs()](caracal/packages/caracal-server/caracal/core/audit.py#L456)
  - Retention period: 2555 days (7 years, default)
  - Identifies logs older than cutoff
  - Recommends export to cold storage before deletion

**Partition-based Lifecycle:**
- [PartitionManager.archive_old_partitions()](caracal/packages/caracal-server/caracal/db/partition_manager.py#L260)
  - Detaches partitions older than `months_to_keep` (default 12)
  - Detached partitions should be backed up to cold storage
  - Dry-run mode available for validation

---

### 7. LEDGER INTEGRITY & VERIFICATION ✓

**Merkle Root Batching:**
- [MerkleRoot](caracal/packages/caracal-server/caracal/db/models.py#L130)
  - `merkle_root` (SHA256 hash), `signature` (ECDSA signature)
  - `first_event_id`, `last_event_id` (batch range)
  - `source` column differentiates: "live" vs "migration" backfill
  - Unique constraint on `batch_id`
  - Indexes: batch_id, (first_event_id, last_event_id)

**Recovery from Snapshots:**
- [RecoveryManager.recover_from_latest_snapshot()](caracal/packages/caracal-server/caracal/merkle/recovery.py#L40)
  - Loads latest snapshot
  - Replays events after snapshot timestamp
  - Validates Merkle roots for post-snapshot batches
  - Returns: events_replayed, duration, Merkle batch validation results

**Batch Timeout Logic:**
- [MerkleBatcher._timeout_handler()](caracal/packages/caracal-server/caracal/merkle/batcher.py#L226)
  - Batch timeout: 300 seconds (configurable)
  - Closes batch automatically after timeout or size threshold (1000 events)
  - Cancellable: timeout task cancelled if batch fills first

---

### 8. EDGE CASES & ERROR HANDLING ✓

**Principal Expiry with TTL Leases:**
- [PrincipalTTLManager](caracal/packages/caracal-server/caracal/identity/principal_ttl.py#L110)
  - Redis-backed TTL tracking: separate "pending_attestation" and "active" stages
  - Pending TTL: attestation window (e.g., 5 minutes)
  - Active TTL: remains after attestation (e.g., 1 hour)
  - Orphaned state cleanup: expired lease removes all metadata + TTL index

**Orphaned State from Attestation Timeout:**
- [PrincipalTTLExpiryProcessor._expire_pending_attestation()](caracal/packages/caracal-server/caracal/identity/principal_ttl.py#L340)
  - Transitions principal: PENDING_ATTESTATION → EXPIRED
  - Revokes active mandates (-undo spawned authority)
  - Records: "attestation_nonce_timeout" event in ledger
  - **Recovers state**: metadata updated with failure reason

**Handoff Token Transaction Failure:**
- [SessionManager.issue_handoff_token()](caracal/packages/caracal-server/caracal/core/session_manager.py#L700)
  - If DB insert fails (exception), source token is **NOT revoked locally**
  - Source scope narrowing only happens on successful DB commit
  - Test: `test_handoff_transaction_failure_leaves_source_scope_unchanged()` confirms source token remains valid
  - **Idempotency**: on retry, transfer already exists → returns cached token

**Revocation State Consistency:**
- [PrincipalRevocationOrchestrator._revoke_single_principal()](caracal/packages/caracal-server/caracal/core/revocation.py#L360)
  - Atomic transaction: principal + mandates + edges revoked together
  - Record authority event: "principal_revoked" in ledger
  - **Cascade semantics**: children revoked via async jobs if >250 descendants
  - Denylist added after DB commit (non-fatal if fails)

**Circuit Breaker Fault Isolation:**
- [CircuitBreaker.call()](caracal/packages/caracal-server/caracal/core/circuit_breaker.py#L155)
  - States: CLOSED (normal) → OPEN (fail-fast) → HALF_OPEN (test recovery)
  - Transition on: failure_threshold exceeded (CLOSED→OPEN)
  - Recovery on: success_threshold exceeded in HALF_OPEN (→CLOSED)
  - Timeout before HALF_OPEN: prevents cascading failures

**Revocation Sync Worker:**
- [RevocationSyncWorker](caracalEnterprise/services/gateway/revocation_sync.py#L110)
  - Background polling: checks for revoked mandates every 30 seconds
  - In-memory cache + DB fallback (single-check authoritative)
  - Listeners notified: policy cache invalidated on revocation
  - **Checkpoint tracking**: only syncs delta since last checkpoint

---

### 9. SENSITIVE DATA LEAKAGE — NO CRITICAL ISSUES FOUND ✓

**Secured:**
- API keys: never logged plaintext, masked in responses ✓
- Secrets: stored in vault, not in DB ✓
- Passwords: redacted in logs ✓
- Tokens: redacted by key pattern ✓

**Verified:**
- CCLE credential tests confirm non-disclosure
- Logging redaction tests pass
- Reveal endpoint enforces 30-second TTL
- Vault audit events track all operations

---

### 10. CRITICAL OBSERVATIONS

| Finding | Status | Evidence |
|---------|--------|----------|
| **Immutability**: Event records never modified | ✓ Implemented | No update triggers on `ledger_events`, `authority_ledger_events`, `audit_logs` |
| **Durability**: PostgreSQL-only, no fallback | ✓ Implemented | [db/connection.py](caracal/packages/caracal-server/caracal/db/connection.py#L1): "only supported backend" |
| **Transactional Scope**: Atomic commits | ✓ Implemented | `session_scope()` context manager with rollback |
| **Audit Trail**: Comprehensive ledger | ✓ Implemented | Authority, metering, admin, session events all logged |
| **Idempotency**: Keys + signatures | ✓ Implemented | HMAC + SHA256 idempotency key with unique constraint |
| **Redaction**: Sensitive fields masked | ✓ Implemented | Structured logging processor strips passwords/tokens |
| **Retention**: TTL-based purge automation | ✓ Implemented | Archive records with `auto_delete_at` + `purge_due` flag |
| **Fault Recovery**: Snapshot-based replay | ✓ Implemented | Merkle snapshots + event replay with integrity validation |
| **Replication Safety**: Nonce via Redis | ✓ Implemented | SETNX with TTL prevents cross-replica replay |
| **Corruption Risk**: Transaction failure handling | ⚠️ Noted | Handoff transfer: source NOT revoked if DB fails (retriable) |

---

### SUMMARY

All core ledger integrity patterns are **correctly implemented and durable**:
- Events stored immutably in PostgreSQL with auto-incrementing IDs
- Audit trails comprehensive across authority, metering, admin actions
- Idempotency enforced via HMAC signatures + unique idempotency keys
- Sensitive data redacted from logs; secrets vault-backed
- Event retention automated with TTL and ledger proof chains
- Transaction safety via session scopes; cascade ordered (leaves-first)
- Recovery supported via snapshots + event replay with Merkle validation
- Edge cases handled: TTL expiry, orphaned state cleanup, partial failures

**No critical state loss, leakage, or replay vulnerabilities detected.**