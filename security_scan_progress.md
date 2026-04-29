# Enterprise vs OSS Security Drift - Scan Complete

## Key Findings Summary (8 Critical + 6 High Severity Issues)

### CRITICAL Issues (Consistent across both, no drift)
- Mandate state validation (revocation, expiration) ✓
- Principal lifecycle status enforcement ✓
- Signature verification on mandates ✓

### HIGH Severity Issues (Drift detected)

1. **SESSION REVOCATION HANDLING INCOMPLETE IN ENTERPRISE** (HIGH)
2. **MFA ENFORCEMENT - ENTERPRISE ONLY** (HIGH)  
3. **PRIVILEGED REAUTH THRESHOLD - ENTERPRISE ONLY** (HIGH)
4. **RBAC NOT IN OSS** (HIGH)
5. **AUTHORITY MANDATE WRAPPING - ENTERPRISE ADDS, OSS LACKS** (HIGH)
6. **CONFIG SECURITY DEFAULTS DIFFER** (MEDIUM)

### Other Notable Differences
- Multi-workspace vs single workspace
- Gateway provider registry enforcement
- API key vs signature authentication
- Feature flags and development mode bypass

Status: 14 distinct issues identified with severity levels, locations, and remediation guidance documented.
