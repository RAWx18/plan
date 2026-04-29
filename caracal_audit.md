# Caracal & CaracalEnterprise Audit - Session Notes

## Key Findings So Far

### OSS (Caracal) Structure
- **Bootstrap**: `caracal bootstrap` creates system principal, issues AIS nonce, auto-grants dev allowlist (dev-only)
- **Principal Registration**: `caracal principal register` - manual CLI command, no approval workflow
- **Policy Creation**: `caracal policy create` - manual CLI command, direct DB insert, no approval workflow
- **Allowlist**: `caracal allowlist create/delete` - manual CLI, no approval gates
- **Vault Setup**: **GAP** - No first-run vault bootstrap command (`caracal vault init` missing per PACKAGING.md §2.4)
- **DB/Alembic**: Migrations require alembic.ini resolution from repo (should be embedded)

### Enterprise (CaracalEnterprise) Structure
- API service on port 9000
- Gateway service for auth (mTLS, JWT, API Key)
- Separate vault service
- Separate DB (port 5433)

### Authorization/Approval Patterns Found
1. **No centralized approval workflow** - all registration/policy ops go direct to DB
2. **Dev mode relaxation**: `_dev_example_mode_enabled()` creates wildcard allowlist (`*`) with glob pattern
3. **Bootstrap auto-grants**: System principal gets `system.admin`, `system.stats.read`, `mcp.tool_registry.manage`
4. **No audit trail for registration**: Principal registration commands don't log to AuditLog table
5. **Manual CLI-only workflows** for registering principals and policies

### Logging/Auditability Gaps
- `AuditLog` table exists (key_audit.py uses it)
- Audit logging for **key lifecycle events** implemented (append_key_audit_event)
- **No audit logging for**:
  - Principal registration
  - Policy creation/modification
  - Allowlist changes
  - Authorization events
  - Manual approval workflows (don't exist)

### Brittle Manual Setup
1. Vault bootstrap missing - must read docs, manually set env vars
2. Alembic.ini lookup walks up from installed module (only works in checkout)
3. Compose file in 3 places - precedence chain unclear
4. SDK imports core modules (should be standalone)
5. Single PyPI package conflates user/server code

### Security Gates That Work
1. PrincipalKind and lifecycle status enums enforced
2. Certificate validation in gateway auth (mTLS)
3. JWT signature verification
4. Redis requires password
5. DB connection requires auth
6. Owner-only file permissions enforced (.env chmod 0o600)
7. Session tokens via AIS Unix socket

### Autonomy Blockers
1. **No auto-provisioning** - principals/policies must be created manually
2. **No delegation auto-approval** - max_network_distance/allow_delegation set once at policy creation
3. **Manual secret refs** - SecretsAdapter requires explicit ref storage
4. **No auto-scaling for allowlist** - max_patterns_per_principal must be enforced at create time
5. **Workspace creation manual** - requires `caracal workspace create` command

## Enterprise (CaracalEnterprise) Findings

### Authorization/Registration Workflows
- **No approval gates** - principal/policy creation is direct to API
- **Enterprise routes** allow:
  - Tenant/workspace creation during onboarding (draft → active)
  - Principal registration via API (goes to Caracal Core)
  - Policy creation via API (direct DB insert)
  - No manual approval review required
  
### Onboarding Flow
- Workspace state machine: `configure` → `team` → `pricing` → `payment` → `workspace` → `complete`
- No manual approval after payment - auto-transitions to workspace creation
- `_compute_workspace_state()` auto-advances steps based on payment status
- **Gap**: No explicit authorization request/approval workflow

### Quota & Tenant Management
- **Tier-based quotas** (starter/professional/enterprise/unlimited)
- **TenantContextMiddleware** enforces quotas via Redis
- **API key caching** (5 min TTL) for validation
- Rate limiting: 60-6000 RPM depending on tier
- **No per-action approval** - quotas are enforcer only

### Compliance & Audit
- **Compliance events** recorded in-memory (max 5000, circular buffer)
- `record_gateway_denial_event()` logs enforcement decisions
- Vault migrations track audit log (MigrationAuditRecord)
- **Gap**: In-memory event buffer not persisted to DB

### Security Gates Working
1. Organization tier validation
2. API key hash lookup + validation
3. JWT workspace claim extraction
4. Gateway denial event recording
5. Quota enforcement (fail-closed on Redis errors)
6. User permission checks (require_page_permission)

### Autonomy Blockers in Enterprise
1. **Manual onboarding** - requires step-by-step progression
2. **License validation** required but entry is automatic
3. **No auto-provisioning** - workspace requires manual transition
4. **No MFA flow** documented - only `require_mfa_for_privileged_actions` flag
5. **No approval workflow** for policy changes
6. **No delegation auto-approval** - set once at creation

### Auditability Gaps Found
1. Principal registration not logged to AuditLog
2. Policy creation/modification not logged
3. Compliance events stored in-memory only (volatile)
4. User session revocation on principal deactivation (Good)
5. No per-action audit trail for allowlist changes
6. No mandate denial audit context linkage

### Missing Automatable Workflows
1. **User onboarding** requires manual email/link progression
2. **Team assignment** requires manual role assignment
3. **Credential rotation** not automated (keys set once at principal creation)
4. **Policy updates** require manual API calls (no delegation of update authority)
5. **Workspace provisioning** manual (after payment, still requires transition)
6. **MFA enforcement** policy-only, no auto-enrollment
7. **Sandbox/dev** mode requires manual env config
