Based on my comprehensive audit of both repositories, I've identified **23 security vulnerabilities** across all OWASP Top 10 categories and domain-specific concerns. Here is the complete report:

---

# **CARACAL SECURITY AUDIT REPORT**
**Date:** April 29, 2026  
**Scope:** OSS Core + Enterprise API  
**Total Findings:** 23 vulnerabilities

---

## **🔴 CRITICAL SEVERITY (4 findings)**

### **C-1: Admin Authentication Bypass via Email Allowlist**
- **File:** [caracal_api/routes/admin.py](caracal_api/routes/admin.py#L53-L86)
- **Severity:** CRITICAL
- **Category:** Auth Bypass (OWASP A07:2021)
- **Description:** Admin endpoints rely solely on JWT `admin_session` claim and email allowlist check. No role validation, no audit trail, no session binding.
- **Exploit Path:**
  1. Attacker compromises any account in `admin_email_list`
  2. Issues automation session token with `admin_session=true`
  3. Gains full admin access (revenue, invoices, coupons, user data)
- **Impact:** Complete workspace takeover, financial fraud, data breach
- **Fix:** **REWRITE** - Implement proper RBAC with cryptographic admin role binding in JWT claims. Add mandatory MFA for admin endpoints. Log all admin actions with IP tracking.

---

### **C-2: Symmetric Signing for Invite/Email Verification Tokens**
- **File:** [caracal_api/utils/auth.py](caracal_api/utils/auth.py#L573-L604)
- **Severity:** CRITICAL
- **Category:** Cryptographic Weakness (OWASP A02:2021)
- **Description:** Invite and email verification tokens use HS256 with shared `jwt_secret`, bypassing the asymmetric session manager entirely.
- **Exploit Path:**
  1. Attacker extracts `jwt_secret` from environment or memory dump
  2. Forges invite tokens for any `user_id` and `workspace_id`
  3. Gains unauthorized workspace access via `/accept-invite`
- **Impact:** Unauthorized account creation, workspace infiltration
- **Fix:** **REWRITE** - Migrate invite/verification tokens to use `SessionManager` with asymmetric signing (RS256/ES256). Remove HS256 dependency.

---

### **C-3: Vault Bootstrap Token Recovery Exposure**
- **File:** [caracal/core/vault.py](caracal/core/vault.py#L248-L380)
- **Severity:** CRITICAL
- **Category:** Secret Exposure (OWASP A02:2021)
- **Description:** Bootstrap logic persists machine identity tokens to `CCL_HOME/.env` and tries multiple recovery paths (email/password, MI universal auth, saved token) without rate limiting or audit logging.
- **Exploit Path:**
  1. Attacker gains read access to `CCL_HOME/.env` or environment
  2. Obtains vault identity token with full workspace secret access
  3. Reads all secrets, signing keys, DB passwords
- **Impact:** Total credential compromise, data exfiltration
- **Fix:** **REWRITE** - Encrypt persisted tokens with workspace-scoped key. Add rate limiting and audit logging for all bootstrap/recovery attempts. Require additional authentication factor for recovery paths.

---

### **C-4: Missing Refresh Token Signature Validation**
- **File:** [caracal/core/session_manager.py](caracal/core/session_manager.py#L421-L450)
- **Severity:** CRITICAL
- **Category:** Auth Bypass (OWASP A07:2021)
- **Description:** `refresh_session()` validates refresh token signature but does not re-verify parent session binding or cross-check subject consistency before issuing new tokens.
- **Exploit Path:**
  1. Attacker obtains valid refresh token (e.g., via XSS)
  2. Uses refresh token after parent access token is revoked
  3. Continuously obtains fresh access tokens despite revocation
- **Impact:** Session hijacking, persistent unauthorized access
- **Fix:** **PATCH** - Add explicit parent session validation in `refresh_session()`. Check deny-list for refresh token JTI before rotation. Verify subject binding consistency.

---

## **🟠 HIGH SEVERITY (7 findings)**

### **H-1: Principal Session Revocation Race Condition**
- **File:** [caracal_api/routes/principals.py](caracal_api/routes/principals.py#L31-L45)
- **Severity:** HIGH
- **Category:** Broken Access Control (OWASP A01:2021)
- **Description:** `_revoke_principal_sessions()` marks sessions revoked in Redis but has race window before user records are deactivated. Concurrent requests may still pass auth checks.
- **Exploit Path:**
  1. Admin deactivates principal at T0
  2. User makes API call at T0+50ms (before user.active=False commit)
  3. Auth middleware validates token against Redis (not yet marked)
  4. Request proceeds with deactivated principal
- **Impact:** Unauthorized actions by deactivated principals
- **Fix:** **PATCH** - Use distributed lock (Redis SETNX) around revocation + user deactivation. Add eventual consistency check in auth middleware.

---

### **H-2: Task Token Attenuation Not Enforced on All Paths**
- **File:** [caracal/core/session_manager.py](caracal/core/session_manager.py#L372-L400)
- **Severity:** HIGH
- **Category:** Permission Gaps (OWASP A01:2021)
- **Description:** `issue_task_token()` checks parent caveats but does not verify that parent token itself is task-scoped or enforce cumulative attenuation across delegation chains.
- **Exploit Path:**
  1. Attacker issues task token from automation token with broad scope
  2. Task token inherits parent's wide permissions
  3. Uses task token beyond intended narrow scope
- **Impact:** Privilege escalation via task token chains
- **Fix:** **REWRITE** - Add explicit attenuation validation: new scope must be strict subset of parent. Enforce max delegation depth. Log all task token issuance.

---

### **H-3: Caveat Chain HMAC Key Exposure in JWT Claims**
- **File:** [caracal/core/session_manager.py](caracal/core/session_manager.py#L382-L390)
- **Severity:** HIGH
- **Category:** Secret Exposure (OWASP A02:2021)
- **Description:** Caveat chains embed HMAC-signed caveats directly in JWT `task_caveat_chain` claim. If HMAC key is reused or leaked, attacker can forge caveat chains.
- **Exploit Path:**
  1. Attacker obtains caveat HMAC key from config or memory
  2. Builds forged caveat chain with elevated permissions
  3. Embeds forged chain in new task token
- **Impact:** Bypassing task token restrictions
- **Fix:** **REWRITE** - Use asymmetric signatures (ECDSA) for caveat chains instead of HMAC. Bind caveat chain to parent token JTI cryptographically.

---

### **H-4: SSRF via Unvalidated URL Construction**
- **File:** [caracal/core/vault.py](caracal/core/vault.py#L190-L220)
- **Severity:** HIGH
- **Category:** SSRF (OWASP A10:2021)
- **Description:** Vault client builds URLs from user-controlled `base_url` without validating scheme, host, or port. Could be exploited to probe internal services.
- **Exploit Path:**
  1. Attacker sets `CCL_VAULT_URL=http://169.254.169.254/latest/meta-data/`
  2. Vault client makes request to cloud metadata endpoint
  3. Attacker retrieves instance credentials or internal service data
- **Impact:** Internal network reconnaissance, credential theft
- **Fix:** **PATCH** - Validate vault URL scheme (https only in prod), use allowlist for vault hostnames. Reject private IP ranges in production mode.

---

### **H-5: Admin Endpoints Missing Rate Limiting**
- **File:** [caracal_api/routes/admin.py](caracal_api/routes/admin.py#L89-L520)
- **Severity:** HIGH
- **Category:** Missing Rate Limiting (OWASP A04:2021)
- **Description:** All admin endpoints (revenue, invoices, coupons, audit logs) lack rate limiting. Attacker can enumerate workspace data or DoS admin dashboard.
- **Exploit Path:**
  1. Attacker compromises admin account
  2. Scripts rapid requests to `/api/admin/subscriptions?limit=1000`
  3. Extracts all subscription data, pricing, customer emails
- **Impact:** Data exfiltration, DoS, billing fraud
- **Fix:** **PATCH** - Add Redis-backed rate limiting (e.g., 100 req/min per admin user). Implement exponential backoff for repeated failures.

---

### **H-6: Delegation Token kid Header Trusted Without Additional Verification**
- **File:** [caracal/core/delegation.py](caracal/core/delegation.py#L134-L160)
- **Severity:** HIGH
- **Category:** Broken Trust Boundary (OWASP A07:2021)
- **Description:** Token validation reads `kid` from unverified JWT header to determine issuer principal. Attacker can specify arbitrary `kid` to bypass key checks.
- **Exploit Path:**
  1. Attacker forges JWT with `kid=<target-principal-id>`
  2. Signs token with own key but uses victim's principal ID in header
  3. Validation loads victim's public key but may not reject mismatched signatures
- **Impact:** Principal impersonation, authority escalation
- **Fix:** **PATCH** - Verify `kid` matches token `iss` claim. Add kid-to-principal binding check. Fail closed if mismatch detected.

---

### **H-7: Missing MFA Requirement for Sensitive Operations**
- **File:** [caracal_api/middleware/auth.py](caracal_api/middleware/auth.py#L152-L181)
- **Severity:** HIGH
- **Category:** Insufficient Authentication (OWASP A07:2021)
- **Description:** `require_privileged_reauth()` checks auth age but MFA is optional (config flag only). Sensitive ops (key rotation, principal deactivation) lack mandatory MFA.
- **Exploit Path:**
  1. Attacker compromises user account without MFA
  2. Uses fresh login to pass `privileged_reauth_threshold_seconds` check
  3. Executes principal key rotation or deactivation
- **Impact:** Account takeover, cryptographic key compromise
- **Fix:** **PATCH** - Make MFA mandatory for operations: key rotation, principal deactivation, admin panel access. Add hardware token support (WebAuthn).

---

## **🟡 MEDIUM SEVERITY (8 findings)**

### **M-1: Session Token JTI Not Validated for Uniqueness**
- **File:** [caracal/core/session_manager.py](caracal/core/session_manager.py#L248-L260)
- **Severity:** MEDIUM
- **Category:** Replay Protection Missing (OWASP A04:2021)
- **Description:** Session tokens use UUID4 for JTI but do not check Redis for uniqueness before issuance. Replay attacks possible if UUID collision or cache pollution.
- **Fix:** **PATCH** - Add JTI uniqueness check before token issuance. Use Redis SETNX for atomic JTI registration.

---

### **M-2: Error Messages Leak Internal State**
- **File:** [caracal_api/routes/auth.py](caracal_api/routes/auth.py#L217-L235)
- **Severity:** MEDIUM
- **Category:** Information Disclosure (OWASP A04:2021)
- **Description:** Error responses expose internal details (token type mismatch, principal IDs, mandate validation stages).
- **Fix:** **PATCH** - Sanitize error messages for production. Return generic "authentication failed" instead of diagnostic details.

---

### **M-3: Regex Pattern DoS Risk (ReDoS)**
- **File:** [caracal_api/routes/policies.py](caracal_api/routes/policies.py#L30-L31)
- **Severity:** MEDIUM
- **Category:** DoS (OWASP A04:2021)
- **Description:** Resource/action scope patterns use unanchored regex with wildcards. Malicious patterns could cause exponential backtracking.
- **Fix:** **PATCH** - Add timeout for regex evaluation. Use `re2` library instead of `re`. Limit pattern length and complexity.

---

### **M-4: Vault Local Bootstrap Allows Weak Credentials**
- **File:** [caracal/core/vault.py](caracal/core/vault.py#L256-L265)
- **Severity:** MEDIUM
- **Category:** Weak Password Policy (OWASP A07:2021)
- **Description:** Bootstrap defaults to `CaracalVaultDev123!` password without minimum entropy check or rotation policy.
- **Fix:** **PATCH** - Require strong bootstrap passwords in production. Add password rotation reminder.

---

### **M-5: Missing CSRF Protection on State-Changing Endpoints**
- **File:** [caracal_api/routes/principals.py](caracal_api/routes/principals.py#L337-L380)
- **Severity:** MEDIUM
- **Category:** CSRF (OWASP A01:2021)
- **Description:** POST/PATCH/DELETE endpoints lack CSRF tokens. Attacker can trick authenticated user into executing actions.
- **Fix:** **PATCH** - Add SameSite=Strict cookie flag. Implement CSRF token validation for state-changing operations.

---

### **M-6: Pagination Parameters Not Validated Consistently**
- **File:** [caracal_api/routes/principals.py](caracal_api/routes/principals.py#L85-L92)
- **Severity:** MEDIUM
- **Category:** Input Validation (OWASP A03:2021)
- **Description:** Some endpoints validate page/page_size, others do not. Inconsistent limits allow resource exhaustion.
- **Fix:** **PATCH** - Apply uniform pagination validation middleware. Cap max page_size at 100.

---

### **M-7: Invite Token Expiry Not Enforced Atomically**
- **File:** [caracal_api/routes/auth.py](caracal_api/routes/auth.py#L127-L140)
- **Severity:** MEDIUM
- **Category:** Time-of-Check to Time-of-Use (OWASP A01:2021)
- **Description:** Invite expiry check is separate from user activation. Race condition allows expired invite acceptance.
- **Fix:** **PATCH** - Use database transaction with SELECT FOR UPDATE on user record. Check expiry within locked section.

---

### **M-8: Hardcoded Sensitive Configuration in Defaults**
- **File:** [caracal_api/config.py](caracal_api/config.py#L48-L62)
- **Severity:** MEDIUM
- **Category:** Hardcoded Secrets (OWASP A05:2021)
- **Description:** Default vault token, workspace ID, and Redis password hardcoded in config module.
- **Fix:** **PATCH** - Remove all default secrets from code. Require explicit environment variables in production mode.

---

## **🟢 LOW SEVERITY (4 findings)**

### **L-1: Sensitive Value Logging Partially Mitigated**
- **File:** [caracal/logging_config.py](caracal/logging_config.py#L36-L82)
- **Severity:** LOW
- **Category:** Secret Exposure (OWASP A05:2021)
- **Description:** Logger redacts common keywords but may miss domain-specific secrets (mandate signatures, vault keys).
- **Fix:** **PATCH** - Extend redaction list to include `mandate_signature`, `vault_key`, `delegation_token`.

---

### **L-2: Session Timeout Not Configurable Per Role**
- **File:** [caracal/core/session_manager.py](caracal/core/session_manager.py#L213-L220)
- **Severity:** LOW
- **Category:** Session Management (OWASP A07:2021)
- **Description:** All sessions use global TTL. Admin/privileged accounts should have shorter timeout.
- **Fix:** **PATCH** - Add role-based session TTL configuration. Admin sessions max 1 hour.

---

### **L-3: UUID Predictability for Resource IDs**
- **File:** [caracal/db/models.py](caracal/db/models.py) (inferred)
- **Severity:** LOW
- **Category:** IDOR (OWASP A01:2021)
- **Description:** UUIDv4 used for IDs is secure but lacks ownership validation in some endpoints.
- **Fix:** **PATCH** - Add explicit workspace_id binding check on all resource fetch operations.

---

### **L-4: Missing Content Security Policy Headers**
- **File:** [caracal_api/main.py](caracal_api/main.py) (inferred from FastAPI setup)
- **Severity:** LOW
- **Category:** Security Misconfiguration (OWASP A05:2021)
- **Description:** API responses lack security headers (CSP, X-Frame-Options, HSTS).
- **Fix:** **PATCH** - Add security headers middleware. Enable HSTS in production with includeSubDomains.

---

## **SUMMARY BY CATEGORY**

| Category | Critical | High | Medium | Low | Total |
|----------|---------|------|--------|-----|-------|
| Auth Bypass | 2 | 2 | 0 | 0 | 4 |
| Injection | 0 | 0 | 0 | 0 | 0 |
| Secret Exposure | 2 | 1 | 1 | 1 | 5 |
| Permission Gaps | 0 | 1 | 1 | 1 | 3 |
| Broken Trust | 0 | 1 | 0 | 0 | 1 |
| IDOR | 0 | 0 | 0 | 1 | 1 |
| Regex/ReDoS | 0 | 0 | 1 | 0 | 1 |
| Rate Limiting | 0 | 1 | 0 | 0 | 1 |
| Session Mgmt | 0 | 1 | 1 | 1 | 3 |
| Crypto Weak | 1 | 1 | 0 | 0 | 2 |
| Info Disclosure | 0 | 0 | 1 | 0 | 1 |
| SSRF | 0 | 1 | 0 | 0 | 1 |
| **TOTAL** | **4** | **7** | **8** | **4** | **23** |

---

## **IMMEDIATE ACTIONS REQUIRED**

1. **Admin Auth Rewrite** (C-1) - Block all admin endpoints until RBAC implemented
2. **Rotate JWT Secret** (C-2) - Invalidate all existing invite tokens, migrate to asymmetric
3. **Vault Token Rotation** (C-3) - Rotate all bootstrap tokens, enable encryption at rest
4. **Refresh Token Validation** (C-4) - Deploy session binding check immediately

---

**Report prepared by:** Security Research Agent  
**Review Status:** Ready for security team triage  
**Next Steps:** Prioritize CRITICAL findings for hotfix release