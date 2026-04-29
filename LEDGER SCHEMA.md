Based on my comprehensive audit of the ledger & state integrity subsystem, here's the detailed assessment:

---

## LEDGER SCHEMA SUMMARY

### Primary Tables
- **`ledger_events`** ([models.py](caracal/packages/caracal-server/caracal/db/models.py#L84-L125)): Metering events for resource usage
  - Columns: `event_id` (PK, auto-increment), `principal_id`, `timestamp`, `resource_type`, `quantity`, `event_metadata` (JSON), `merkle_root_id` (FK)
  - Index: `ix_ledger_events_agent_timestamp` (principal_id, timestamp)

- **`authority_ledger_events`** ([models.py](caracal/packages/caracal-server/caracal/db/models.py#L745-L840)): Authority decision log
  - Columns: `event_id` (PK, auto-increment), `event_type` (issued/validated/denied/revoked), `principal_id`, `mandate_id`, `decision`, `denial_reason`, `requested_action`, `requested_resource`, `correlation_id`, `merkle_root_id`
  - Child table: `authority_event_attributes` (typed key-value pairs via `_encode_authority_attribute` / `_decode_authority_attribute`)
  - Indexes: (principal_id, timestamp), (mandate_id, timestamp)

- **`merkle_roots`** ([models.py](caracal/packages/caracal-server/caracal/db/models.py#L184-L220)): Signed batch checksums
  - Columns: `root_id`, `batch_id`, `merkle_root` (hex SHA-256), `signature` (hex ECDSA), `event_count`, `first_event_id`, `last_event_id`, `source` ("live"|"migration"), `created_at`
  - Index: `ix_merkle_roots_event_range` (first_event_id, last_event_id)

- **`ledger_snapshots`** ([models.py](caracal/packages/caracal-server/caracal/db/models.py#L222-L244)): Recovery checkpoints

---

## INTEGRITY VERDICT BY DIMENSION

| Dimension | Status | Notes |
|-----------|--------|-------|
| **1. Append-only enforcement** | **WEAK** | No database-level triggers/constraints prevent UPDATE/DELETE; relies on code discipline |
| **2. Merkle integrity** | **PASS** | Proper batch hashing, ECDSA P-256 signing, verification path present |
| **3. Completeness** | **WEAK** | Ledger write is optional; failure doesn't block authority decisions |
| **4. Sensitive data in ledger** | **WEAK** | `event_metadata` accepts arbitrary dicts; no sanitization of intent parameters |
| **5. Compactness** | **PASS** | Only decisions logged, not every read; focused on meaningful events |
| **6. Inspection ergonomics** | **PASS** | CLI query tool + REST API + correlation_id support |
| **7. Indexing for queries** | **PASS** | Composite indexes on (principal_id, timestamp), (mandate_id, timestamp), correlation_id index |
| **8. Retention/pruning** | **FAIL** | No TTL, archival, or partitioning strategy visible; infinite growth risk |
| **9. Correlation tracking** | **PASS** | `correlation_id` column supported throughout; request tracing enabled |
| **10. Tamper resistance** | **PASS** | ECDSA signatures on Merkle roots; cryptographic integrity chain |
| **11. Failure mode** | **FAIL** | Fail-open: ledger write failure doesn't block authority decision |

---

## DETAILED FINDINGS

### 1. **Append-Only Enforcement: WEAK**

**Finding**: No database-level constraints prevent UPDATE or DELETE on ledger tables.

**Evidence**:
- [models.py L84-87](caracal/packages/caracal-server/caracal/db/models.py#L84-L87): Documentation claims "immutable" but provides no enforcement
- No PostgreSQL RULE, TRIGGER, or CHECK constraints found in migrations
- No `sa.CheckConstraint` in ORM definitions
- Grep for UPDATE/DELETE on ledger tables returns only comments, not constraints

**Risk**: A malicious DBA or compromised application code path could silently tamper with historical records.

**Recommendation**: 
```sql
CREATE RULE prevent_ledger_update AS ON UPDATE TO ledger_events DO INSTEAD NOTHING;
CREATE RULE prevent_ledger_delete AS ON DELETE TO ledger_events DO INSTEAD NOTHING;
CREATE RULE prevent_authority_ledger_update AS ON UPDATE TO authority_ledger_events DO INSTEAD NOTHING;
CREATE RULE prevent_authority_ledger_delete AS ON DELETE TO authority_ledger_events DO INSTEAD NOTHING;
```

---

### 2. **Merkle Integrity: PASS**

**Finding**: Proper cryptographic hashing and signing chain established.

**Merkle Tree Construction** ([merkle/tree.py](caracal/packages/caracal-server/caracal/merkle/tree.py#L1-L100)):
- SHA-256 leaf hashing with binary Merkle tree
- Builder pattern supports batch construction

**Signing** ([merkle/signer.py](caracal/packages/caracal-server/caracal/merkle/signer.py#L31-L77)):
- ECDSA P-256 (ES256) signatures on root hashes
- `MerkleRootSignature` dataclass captures: `root_id`, `merkle_root`, `signature`, `batch_id`, event range
- Software signer stores signature in `MerkleRoot.signature` (hex-encoded)

**Verification** ([merkle/verifier.py](caracal/packages/caracal-server/caracal/merkle/verifier.py#L115-L170)):
- Batch verification: recomputes root from events, compares with stored value
- Time-range verification supported
- Migration batches (v0.2 backfill) handled with source tracking

**Gap**: Who holds the signing key? No visible key custody model for Merkle signing keys in the audit.

---

### 3. **Completeness: WEAK**

**Finding**: Authority decisions ARE logged, but ledger write failures don't block decisions (fail-open).

**Log Coverage**:
- ✅ `issue_mandate()` → `record_issuance()` ([authority_ledger.py L50-100](caracal/packages/caracal-server/caracal/core/authority_ledger.py#L50-L100))
- ✅ `validate_mandate()` → `record_validation()` ([authority_ledger.py L114-186](caracal/packages/caracal-server/caracal/core/authority_ledger.py#L114-L186))
- ✅ `revoke_mandate()` → `record_revocation()` ([authority_ledger.py L195-254](caracal/packages/caracal-server/caracal/core/authority_ledger.py#L195-L254))

**Failure Handling** ([authority.py L366-379](caracal/packages/caracal-server/caracal/core/authority.py#L366-L379)):
```python
if self.ledger_writer:
    try:
        self.ledger_writer.record_validation(...)
    except Exception as e:
        logger.error(f"Failed to record ledger event: {e}", exc_info=True)
else:
    logger.debug(f"No ledger writer configured, skipping event recording...")
```
**Problem**: If ledger write fails, decision still executes. No re-queue or circuit breaker.

**Recommendation**: 
- Make ledger write mandatory (fail-closed)
- Implement reliable queue (e.g., Kafka) for async logging with guaranteed delivery
- If queue is full/unavailable, fail authority decision

---

### 4. **Sensitive Data in Ledger: WEAK**

**Finding**: `event_metadata` field accepts arbitrary dictionaries; no sanitization of intent parameters.

**Evidence**:
- [authority_ledger.py L94](caracal/packages/caracal-server/caracal/core/authority_ledger.py#L94): `event_metadata=metadata` passed without filtering
- [authority.py L413](caracal/packages/caracal-server/caracal/core/authority.py#L413): Metadata dict passed directly from caller
- Encode/decode functions ([models.py L861-889](caracal/packages/caracal-server/caracal/db/models.py#L861-L889)) store raw JSON values without filtering

**Risk**: Callers could inadvertently log:
- Intent parameters containing secrets (API keys, tokens, passwords)
- Raw scope specifications with sensitive resource names
- Credential material in decision context

**Recommendation**:
- Implement an allowlist of safe metadata fields (e.g., `decision_ms`, `retry_count`, `policy_version`)
- Scrub metadata before recording:
  ```python
  SAFE_METADATA_KEYS = {
      "decision_latency_ms", "policy_version", "boundary_stage", 
      "delegation_depth", "caveat_chain_length"
  }
  sanitized = {k: v for k, v in (metadata or {}).items() if k in SAFE_METADATA_KEYS}
  ```

---

### 5. **Compactness: PASS**

**Finding**: Ledger focuses on meaningful decisions, not every read.

**What IS logged**:
- Mandate issuance (one event per mandate)
- Validation attempts (one event per validation call)
- Revocations (one event per revocation)

**What is NOT logged**:
- Cache hits on mandate validation
- Policy reads during evaluation
- Delegation graph traversals

This is appropriate for an audit trail.

---

### 6. **Inspection Ergonomics: PASS**

**Query APIs**:
- CLI: `caracal ledger query [--principal-id] [--mandate-id] [--event-type] [--start] [--end] [--format json|table]` ([cli/authority_ledger.py](caracal/packages/caracal-server/caracal/cli/authority_ledger.py#L1-L150))
- REST: `GET /api/ledger/events` with filters (principal_id, event_type, decision, q, start_time, end_time) ([routes/ledger.py](caracalEnterprise/services/api/src/caracal_api/routes/ledger.py#L140-L175))
- Export: CSV export supported
- Correlation: `correlation_id` index enables request tracing from session → mandate validation → ledger entry

**Example query by principal**:
```bash
caracal ledger query --principal-id 550e8400-e29b-41d4-a716-446655440000 --start 2026-01-01
```

---

### 7. **Indexing for Queries: PASS**

**Indexes Present**:
- `ix_ledger_events_agent_timestamp` (principal_id, timestamp) - efficient principal-timeline queries
- `ix_authority_ledger_events_principal_timestamp` (principal_id, timestamp)
- `ix_authority_ledger_events_mandate_timestamp` (mandate_id, timestamp)
- `ix_authority_ledger_events_correlation` (correlation_id, timestamp) - tracing support
- `ix_merkle_roots_event_range` (first_event_id, last_event_id) - batch lookups

**Missing**: Direct index on `event_type` or `decision` for bulk filtering by event kind (currently requires table scan if combined with other filters).

---

### 8. **Retention/Pruning: FAIL**

**Finding**: No archival, TTL, or partition lifecycle strategy visible.

**What exists**:
- Partition manager for `ledger_events` and `authority_ledger_events` by month ([partition_manager.py](caracal/packages/caracal-server/caracal/db/partition_manager.py#L1-L50), [authority_partition_manager.py](caracal/packages/caracal-server/caracal/db/authority_partition_manager.py#L1-L60))
- v0.2 backfill support with "migration" source marking

**What's missing**:
- No DROP/DETACH partition policy after TTL
- No archival to cold storage (S3, GCS)
- No snapshot-based pruning (e.g., "keep detailed events for 90 days, aggregate-only for 1 year")
- No documented retention policy

**Risk**: Database grows unbounded; query performance degrades over time.

**Recommendation**:
- Define retention tiers:
  - **Hot**: 90 days, detailed records, full indexes
  - **Warm**: 1 year, aggregate-only via snapshots, indexes removed
  - **Cold**: Archive to S3 with Merkle root links for verification
- Automate partition detach/archive via Airflow or cron:
  ```sql
  ALTER TABLE authority_ledger_events DETACH PARTITION authority_ledger_events_2024_01;
  COPY authority_ledger_events_2024_01 TO 's3://archive-bucket/auth-ledger-2024-01.parquet';
  ```

---

### 9. **Correlation Tracking: PASS**

**Finding**: Request tracing via `correlation_id` is properly supported.

**Chain**:
1. Session issues JWT with `jti` (session ID)
2. Request middleware attaches `correlation_id` (from header or generates new UUID)
3. Authority validation captures `correlation_id` in `AuthorityLedgerEvent`
4. Ledger query can filter by `correlation_id` to reconstruct full request path

**Evidence**:
- [authority_ledger.py L65, 141, 225](caracal/packages/caracal-server/caracal/core/authority_ledger.py#L65-L226): `correlation_id` parameter accepted
- [models.py L778](caracal/packages/caracal-server/caracal/db/models.py#L778): `correlation_id` column with index
- [routes/ledger.py](caracalEnterprise/services/api/src/caracal_api/routes/ledger.py): REST API supports correlation filter

---

### 10. **Tamper Resistance: PASS**

**Signature Chain**:
- Merkle roots signed with ECDSA P-256
- Signature stored in hex in `MerkleRoot.signature` column
- Verification checks signature validity + root hash recomputation ([verifier.py](caracal/packages/caracal-server/caracal/merkle/verifier.py#L115-L170))

**Gaps**:
- ⚠️ Signing key custody model not audited (assumed Vault-backed per architecture)
- ⚠️ No timestamp authority (TSA) integration; reliance on `created_at` timestamp in application database (could be manipulated if DB is compromised)

---

### 11. **Failure Mode: FAIL**

**Finding**: Ledger write failure is non-fatal; authority decision proceeds (fail-open).

**Code** ([authority.py L366-379](caracal/packages/caracal-server/caracal/core/authority.py#L366-L379)):
```python
if self.ledger_writer:
    try:
        self.ledger_writer.record_validation(...)
    except Exception as e:
        logger.error(f"Failed to record ledger event: {e}", exc_info=True)
```

**Scenario**: If PostgreSQL crashes or ledger write times out:
1. Decision still allowed/denied
2. Audit record is lost
3. No retry, no escalation
4. Caller is unaware

**Recommendation**: Implement fail-closed strategy
- Use async reliable queue (Kafka, RabbitMQ, or PostgreSQL NOTIFY)
- If queue is unavailable, fail authority decision (return HTTP 503)
- Add circuit breaker with exponential backoff

---

## GAPS: Decisions NOT Recorded

Based on code review, the following decisions/events are **NOT** logged to ledger:

1. **Policy Updates**: When an `AuthorityPolicy` is created/modified, no ledger event
2. **Principal Lifecycle Changes**: When principal status changes (ACTIVE → SUSPENDED), not logged
3. **Delegation Edge Creation/Revocation**: DelegationEdgeModel changes not logged
4. **Credential/Key Rotations**: Principal key updates not logged
5. **Caveat Chain Violations**: When caveat validation fails, decision logged but not the caveat itself

**Recommendation**: Add ledger entries for policy/principal/edge changes to create comprehensive audit trail.

---

## SENSITIVE DATA LEAKS: Confirmed Issues

### Risk 1: Intent Parameters in Metadata
**Path**: [authority.py L413](caracal/packages/caracal-server/caracal/core/authority.py#L413)
```python
metadata={
    "intent_parameters": intent.parameters,  # ⚠️ Could include secrets
    "intent_action": intent.action,
    "intent_resource": intent.resource,
}
```

**Example leak**:
```python
intent = Intent(
    action="create_resource",
    resource="database:mysql:prod",
    parameters={"password": "secret123", "host": "db.internal"}
)
# Ledger record now contains plaintext password
```

### Risk 2: Request Body in Metadata
**Pattern**: Callers could pass request body as metadata
```python
metadata={"full_request": request.body}  # ⚠️ Headers, tokens, etc.
```

### Risk 3: Scope Specifications
Scope strings may include sensitive resource identifiers:
```
provider:internal-api:resource:customer-pii:segment:vip-customers
```

---

## INSPECTION UX ASSESSMENT

| Aspect | Status | Detail |
|--------|--------|--------|
| **Query Performance** | GOOD | Indexes on principal_id + timestamp enable O(log n) lookups |
| **Result Clarity** | GOOD | Event type, decision, denial_reason clearly presented |
| **Export** | GOOD | CSV export supported; JSON available via API |
| **Filtering** | GOOD | Time range, principal, event type, correlation_id |
| **Tracing** | GOOD | correlation_id enables request reconstruction |
| **Scalability** | WEAK | No pagination in CLI; API has max 1000 results |
| **Visibility** | WEAK | No aggregation views (e.g., denials per day) |

**Missing**: SQL views for common queries
```sql
-- Denials in last 24h by policy
CREATE VIEW authority_denials_24h AS
SELECT 
  event_type, denial_reason, COUNT(*) as count,
  MIN(timestamp) as first_denial, MAX(timestamp) as last_denial
FROM authority_ledger_events
WHERE event_type = 'denied' AND timestamp > NOW() - INTERVAL '24 hours'
GROUP BY event_type, denial_reason
ORDER BY count DESC;
```

---

## CONCRETE FIXES (Priority Order)

### P0: Critical
1. **Enforce append-only at database level**
   - Add PostgreSQL RULE or CHECK constraints preventing UPDATE/DELETE
   - Add migration

2. **Fail-closed on ledger write failure**
   - Change authority.py to fail if ledger write fails
   - Add circuit breaker for ledger service

3. **Sanitize metadata**
   - Implement allowlist for safe metadata keys
   - Remove intent parameters before logging

### P1: High
4. **Add retention/archival strategy**
   - Document TTL policy (e.g., 90 days hot, 1 year warm)
   - Implement partition lifecycle automation

5. **Add index on event_type and decision**
   - Improves bulk reporting queries

6. **Log policy/principal/edge changes**
   - Extend ledger to cover lifecycle events

### P2: Medium
7. **Add Merkle key rotation policy**
   - Document signing key custody, rotation frequency

8. **Create SQL aggregation views**
   - Support quick compliance reporting (denials, validations by period)

9. **Add pagination to CLI**
   - Limit default result set

---

## SUMMARY

The ledger subsystem has **strong cryptographic integrity** (Merkle trees + ECDSA signatures) but **weak operational safeguards** (no append-only DB constraints, fail-open on write failures, unbounded growth). Metadata sanitization is needed to prevent sensitive data leaks. Correlat...ion ID support is solid. The system is production-ready for basic auditing but requires fixes for compliance-grade immutability.

**Recommended Maturity Path**:
- Phase 1 (Now): P0 fixes + sanitization
- Phase 2 (Q2): Retention/archival + reporting views
- Phase 3 (Q3): Formal immutability certification (AICPA audit trail standards)