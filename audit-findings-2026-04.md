Caracal autonomy/security audit — verified findings (2026-04-29)

CRITICAL fail-open / bypass items confirmed in code:
- mandate.py:288 — `enforce_issuer_policy=False` skips ALL AuthorityPolicy checks
- broker.py:123,1097 — `enforce_scoped_requests` default False; unscoped requests bypass scope contract
- authority.py:559 — `_validate_issuer_authority` does NOT check issuer.lifecycle_status (only subject is checked at ~766)
- authority.py:366-379 — ledger write wrapped in try/except; decision proceeds on failure
- authority.py:632 — mandate.intent_hash stored but never validated against request intent in any stage
- delegation_graph.py:320 — empty scope arrays vacuously covered (allow-all trapdoor)
- models.py:277 — Principal.name has GLOBAL unique constraint, not (owner,name)
- bootstrap.py:155 — falls back to first principal if 'system' missing
- caracalEnterprise auth_context.py:66 — only `is_admin` claim read; documented (admin && admin_session && kind=automation) triple NOT enforced; no DB re-fetch
- gateway_client.py:106/130/151 — naive vs aware datetime mismatches
- Optional dependencies (delegation_graph, deny-list, ledger_writer, revocation publisher) silently weaken validation if not wired

REFUTED / non-issues:
- revocation.py:140 — root principal IS revoked synchronously even when cascade size >250 (sync_targets=[ordered_ids[-1]])
- caveat HMAC key empty raises CaveatChainError (not bypass)
- mandate signature verification is mandatory; no unsigned path
- workspace isolation enforced via Principal.owner filter

Industry alignment to consider:
- SPIFFE SVID for workload attestation (replace nonce ceremony)
- RFC 8693 act/may_act claims for delegation in JWT
- DB-level append-only (PG RULE) for ledger compliance
