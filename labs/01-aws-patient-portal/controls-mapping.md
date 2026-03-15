# Controls Mapping: Threat-to-Control Alignment

This document maps the 10 threats from the threat model to specific security controls and validation steps.

---

## Threat 1: Staff User Credential Compromise (Spoofing)

**Threat:** Staff credentials stolen via phishing; attacker logs in as legitimate staff.

**Controls:**
- Multi-Factor Authentication (MFA) for all staff users (TOTP preferred, SMS fallback)
- Strong password policy: minimum 12 characters, uppercase/lowercase/numbers/symbols required
- Account lockout after 5 failed login attempts (15-minute lockout duration)
- CloudTrail logs all login events (success and failure)
- CloudWatch alerts for failed login attempts (trigger after 3 consecutive failures)
- Session management: force logout after 1 hour of inactivity
- Impossible travel detection: alert if login from two geographically distant locations within 5 minutes

**Validation & Testing:**
- [ ] Attempt login without MFA; confirm rejection
- [ ] Attempt login with invalid MFA code; confirm lockout after 5 attempts
- [ ] Verify CloudTrail captures login events (timestamp, user ID, source IP)
- [ ] Trigger CloudWatch alert by failing login 3 times; confirm SNS notification sent
- [ ] Verify inactivity timeout: create session, wait 1 hour, attempt API call (must fail)
- [ ] Test impossible travel: login from US, attempt login from EU within 1 minute (must trigger alert)

---

## Threat 2: JWT Token Forgery (Tampering)

**Threat:** Attacker crafts fake JWT token or modifies token claims to impersonate another user.

**Controls:**
- RS256 signature validation: API Gateway verifies signature against Cognito public key
- Issuer claim validation: token `iss` must match Cognito pool ID
- Audience claim validation: token `aud` must match API Gateway app client ID
- Token expiry validation: reject tokens where `exp < current_timestamp`
- Lambda secondary validation: verify `sub` (user ID) and `cognito:groups` match expected values
- Token refresh rotation: new refresh token issued on each refresh (limits exposure window)

**Validation & Testing:**
- [ ] Attempt to modify JWT claim (e.g., change `sub` to another user); verify signature validation fails
- [ ] Attempt to forge JWT with invalid signature; confirm API Gateway rejects with 401 Unauthorized
- [ ] Test with expired token (exp timestamp in past); confirm rejection
- [ ] Verify issuer mismatch detected: use token from different Cognito pool (must fail)
- [ ] Verify audience mismatch detected: use token from different app client (must fail)
- [ ] Extract valid token, modify single character in signature; confirm rejection
- [ ] Create token with future `exp` time; verify acceptance
- [ ] Test token claim extraction in Lambda: verify `sub` and `cognito:groups` available in event context

---

## Threat 3: Man-in-the-Middle (MITM) / Request Tampering

**Threat:** Attacker intercepts HTTPS traffic, modifies requests, or performs SSL stripping attack.

**Controls:**
- CloudFront enforces TLS 1.2 minimum (disable SSL 3.0, TLS 1.0, 1.1)
- API Gateway enforces HTTPS-only (no HTTP)
- HSTS (HTTP Strict-Transport-Security) header: set `Strict-Transport-Security: max-age=31536000` on all responses
- AWS WAF blocks HTTP requests, allows HTTPS only
- Request validation: API Gateway JSON schema validation
- Lambda input validation: strict schema checks, reject unexpected fields

**Validation & Testing:**
- [ ] Verify CloudFront security policy: TLS 1.2 minimum (use nmap or similar)
- [ ] Attempt HTTP request to API endpoint; confirm 403/redirect to HTTPS
- [ ] Verify HSTS header present in response (`Strict-Transport-Security: max-age=...`)
- [ ] Test request tampering: modify JSON body in transit, verify signature validation fails (if applicable)
- [ ] Send invalid JSON to API endpoint; confirm 400 Bad Request
- [ ] Send request with extra unexpected fields; confirm API Gateway schema rejects (if strict mode enabled)
- [ ] Send malformed request headers; confirm rejection
- [ ] Verify WAF rule blocks non-HTTPS traffic

---

## Threat 4: NoSQL Injection via DynamoDB

**Threat:** Attacker crafts malicious JSON payload to execute unauthorized DynamoDB operations.

**Controls:**
- Parameterized queries: use boto3 `Key()`, `Attr()`, and `update_expression` (never string concatenation)
- Input validation: strict JSON schema in API Gateway and Lambda
- IAM least-privilege: Lambda role cannot execute arbitrary Scan/Query operations
- DynamoDB query logging: CloudTrail logs all GetItem, Query, Scan operations
- Lambda code review: SAST tools (e.g., Bandit, SonarQube) detect string concatenation in queries

**Validation & Testing:**
- [ ] Attempt DynamoDB injection: send payload like `{"email": {"$ne": null}}`; confirm rejection or no effect
- [ ] Verify boto3 code uses `update_expression` instead of string concatenation (code review)
- [ ] Test parameterized query: verify attacker cannot modify query logic
- [ ] Attempt unauthorized Scan operation from Lambda; confirm IAM denies (403)
- [ ] Verify CloudTrail logs all DynamoDB operations with user identity
- [ ] Run SAST tool on Lambda code; confirm no string concatenation in queries detected
- [ ] Send special characters in input (e.g., `'; DROP TABLE;`, `<script>alert()</script>`); confirm safe handling

---

## Threat 5: Repudiation (Denial of Action)

**Threat:** User claims they did not perform an action; no audit trail to prove otherwise.

**Controls:**
- CloudTrail logs all AWS API calls: Put, Update, Delete operations on DynamoDB, IAM, etc. (includes user identity, timestamp, source IP)
- CloudWatch Logs capture all Lambda invocations with timestamp and user ID
- Application-level audit log: Lambda logs action, user, resource, timestamp, outcome
- S3 MFA Delete: enable on audit log bucket to prevent deletion by unauthorized users
- S3 Object Lock: set COMPLIANCE mode to prevent overwrite or deletion of audit logs
- Immutable log export: daily export of logs to S3 (Glacier) with tamper-evident format (e.g., Merkle tree root)

**Validation & Testing:**
- [ ] Perform action (e.g., UpdatePatientRecord), verify CloudTrail captures event within 15 minutes
- [ ] Verify CloudTrail log includes: user ID, timestamp, action, source IP, request parameters
- [ ] Check CloudWatch Logs for corresponding Lambda invocation
- [ ] Verify S3 audit log bucket has MFA Delete enabled: attempt delete without MFA (must fail)
- [ ] Verify S3 Object Lock COMPLIANCE mode: attempt to delete/overwrite old log (must fail for retention period)
- [ ] Generate audit report from logs; verify chronological order and no gaps
- [ ] Test log export to Glacier; verify immutability

---

## Threat 6: Elevation of Privilege (Overpermissioned Access)

**Threat:** Attacker gains access to resources beyond their authorization level (e.g., patient accesses staff data).

**Controls:**
- Cognito user groups: staff users assigned to `staff` group, patients to `patient` group
- Lambda authorization: verify `cognito:groups` claim, reject unauthorized groups
- IAM least-privilege: Lambda role for patient API has read-only access; staff API role has read/write
- Resource-based policy: DynamoDB table policy restricts access based on user ID (partition key)
- API Gateway resource policy: deny cross-tenant requests (if multi-tenant)

**Validation & Testing:**
- [ ] Create patient user, attempt to call staff-only endpoint (e.g., `/admin/all-patients`); confirm 403 Forbidden
- [ ] Verify Lambda checks `cognito:groups` before allowing action
- [ ] Attempt to modify JWT to add `staff` group; verify signature validation fails
- [ ] Extract patient JWT, attempt to access staff API; confirm rejection at Lambda authorization layer
- [ ] Verify Lambda independently validates group (not relying on JWT alone)
- [ ] Test DynamoDB item-level access: patient can only read their own records (partition key = user ID)
- [ ] Attempt cross-tenant access (if multi-tenant); confirm denied by API Gateway resource policy

---

## Threat 7: Information Disclosure (PHI Leakage via Logs)

**Threat:** PHI (patient name, medical records, medications) accidentally logged and exposed to attackers with log access.

**Controls:**
- CloudWatch Logs redaction: use metric filter patterns to redact PII (e.g., regex replace phone numbers)
- API Gateway access logs exclude request/response body
- Lambda avoid logging full events (use structured logging, log only safe fields)
- X-Ray tracing disabled for sensitive operations (or configured to redact)
- Log bucket encryption: KMS encryption at rest for S3 audit logs
- Log access control: bucket policy restricts read to authorized roles only
- Regular log audits: quarterly search for accidental PHI leakage

**Validation & Testing:**
- [ ] Send API request with PHI in body; verify CloudWatch Logs do NOT contain PHI
- [ ] Lambda logs action; verify logs contain only user ID, timestamp, action (not patient data)
- [ ] Verify API Gateway access log does not include request/response bodies
- [ ] Enable X-Ray tracing; verify sensitive data filtered (or tracing disabled for sensitive Lambda)
- [ ] Verify S3 log bucket encrypted with KMS; confirm plaintext not visible
- [ ] Test log bucket access: attempt read as unauthorized user (must fail)
- [ ] Search CloudWatch Logs for sample PHI (e.g., phone number pattern); confirm zero results
- [ ] Review log retention policy: verify old logs archived to secure storage

---

## Threat 8: Denial of Service (Resource Exhaustion)

**Threat:** Attacker sends millions of requests to overwhelm API, exhaust Lambda concurrency, or drain DynamoDB capacity.

**Controls:**
- AWS WAF rate-based rules: block IPs making >2000 requests per 5-minute window
- API Gateway throttling: 10,000 req/sec burst, 5,000 req/sec sustained limit
- Lambda reserved concurrency: set per-function limit (e.g., 100 concurrent executions)
- Lambda timeout: 30-second timeout to kill stuck requests
- DynamoDB on-demand scaling: auto-scale capacity, but monitor for cost spike
- CloudWatch alarms: trigger on throttling, high Lambda duration, WAF blocked requests
- Auto-recovery: API Gateway returns 429 Too Many Requests; client should implement exponential backoff

**Validation & Testing:**
- [ ] Simulate traffic spike with Apache Bench: `ab -n 100000 -c 1000 https://api.example.com/health`
- [ ] Verify WAF blocks IPs after 2000 req/5min (check WAF logs)
- [ ] Verify API Gateway returns 429 after burst limit exceeded
- [ ] Monitor Lambda CloudWatch metrics during attack (Duration, Throttled Requests, Invocations)
- [ ] Verify Lambda concurrency limit prevents excessive scaling (confirm max = reserved concurrency)
- [ ] Check DynamoDB metrics: confirm throttling after capacity exhausted
- [ ] Verify CloudWatch alarms trigger (SNS notification sent)
- [ ] Test recovery: stop attack, verify system returns to normal within 5 minutes
- [ ] Confirm API Gateway recovers quickly (no lingering throttling)

---

## Threat 9: Privilege Escalation (Patient → Staff)

**Threat:** Patient modifies JWT or exploits Lambda authorization flaw to access staff-only records.

**Controls:**
- Cognito immutable groups: patient assigned to `patient` group at account creation; group cannot be modified by user
- Lambda authorization: explicit check of `cognito:groups` with strict comparison (type-safe UUID check)
- API Gateway resource policy: deny requests without valid `staff` group
- DynamoDB partition key: enforce user-scoped access (patient can only query `partition_key = user_id`)
- IAM role: Lambda role explicitly denies `dynamodb:Scan` (prevents reading all records)
- Code review: SAST tools detect missing authorization checks
- Security testing: explicit test cases for privilege escalation vectors

**Validation & Testing:**
- [ ] Attempt to modify JWT claim `cognito:groups` from `patient` to `staff`; confirm signature validation fails
- [ ] Use valid patient JWT to call staff API; verify Lambda authorization rejects
- [ ] Verify Lambda performs type-safe comparison of group (not string equality vulnerable to case mismatch)
- [ ] Attempt DynamoDB Scan as patient; confirm IAM role denies (403)
- [ ] Verify API Gateway resource policy rejects non-staff requests
- [ ] Test DynamoDB query: patient can only read partition_key = their user ID (verify no access to other partitions)
- [ ] Code review: verify Lambda performs authorization before database query (not after)
- [ ] SAST scan: confirm no missing authorization checks detected

---

## Threat 10: Vulnerable Dependencies (Known CVE Exploitation)

**Threat:** Lambda uses outdated library with known RCE vulnerability; attacker exploits CVE to execute arbitrary code.

**Controls:**
- AWS managed runtimes: Python 3.12, Node.js 20 (auto-patched monthly by AWS)
- Dependency scanning: run OWASP Dependency-Check, Snyk, or AWS CodeArtifact on all deployments
- Lock dependency versions: `requirements.txt` pins exact versions (e.g., `boto3==1.26.0`, not `boto3>=1.20`)
- Quarterly vulnerability scan: scan all deployed Lambda functions for CVEs
- SAST scanning: SonarQube, Bandit, Semgrep detects unsafe code patterns (e.g., eval, exec, SQL injection)
- Least-privilege Lambda role: Lambda cannot access other services/buckets (limits blast radius if code compromised)
- Lambda function URL disabled: disable public access unless explicitly needed

**Validation & Testing:**
- [ ] Run `pip-audit` on requirements.txt; confirm no CVEs detected
- [ ] Run OWASP Dependency-Check; verify report includes all dependencies and no high-risk CVEs
- [ ] Manually verify boto3 version in Lambda: `python -c "import boto3; print(boto3.__version__)"`
- [ ] Run Bandit SAST scan on Lambda code; confirm no unsafe patterns (eval, exec, etc.)
- [ ] Verify Lambda role cannot assume other roles or access S3 (test with `aws s3 ls` - should fail)
- [ ] Disable Lambda function URL (or verify enabled only if intentional)
- [ ] Test dependency update workflow: update vulnerable dependency, deploy, verify fix in CloudWatch Logs
- [ ] Simulate CVE exploitation attempt: if CVE allows RCE, confirm Lambda cannot exfiltrate data (blocked by IAM)

---

## Control Effectiveness Summary

| Control | Impact | Effort | Priority |
|---------|--------|--------|----------|
| MFA for staff | High (prevents credential-based attacks) | Low (built into Cognito) | P0 |
| JWT signature validation | High (prevents token forgery) | Low (API Gateway managed) | P0 |
| Lambda authorization layer | High (prevents privilege escalation) | Medium (custom code needed) | P0 |
| AWS WAF rate limiting | High (prevents DoS) | Low (AWS WAF rules) | P0 |
| CloudTrail auditing | High (repudiation control) | Low (API-level logging) | P0 |
| PHI log redaction | High (prevents data disclosure) | Medium (Lambda logging patterns) | P0 |
| Least-privilege IAM | Medium (containment) | Medium (role management) | P1 |
| DynamoDB encryption | Medium (data at rest) | Low (KMS enabled) | P1 |
| Dependency scanning | Medium (prevents CVE exploitation) | Low (CI/CD integration) | P1 |
| Account lockout | Low (UX impact, security benefit) | Low (Cognito policy) | P2 |

---

## Validation Checklist (30 Tests)

### Authentication & Authorization (8 tests)
- [ ] Test 1: MFA bypass attempts (reject all)
- [ ] Test 2: Invalid JWT token (reject all)
- [ ] Test 3: Expired JWT token (reject all)
- [ ] Test 4: Forged JWT signature (reject all)
- [ ] Test 5: Patient escalation to staff (reject all)
- [ ] Test 6: Modify JWT claims (fail signature validation)
- [ ] Test 7: Lambda authorization function logic (code review)
- [ ] Test 8: Cognito group membership verification (immutable)

### Data Access & DynamoDB (5 tests)
- [ ] Test 9: Patient reads own record (allow)
- [ ] Test 10: Patient reads another patient's record (deny)
- [ ] Test 11: NoSQL injection attempt (deny)
- [ ] Test 12: Unauthorized Scan operation (deny)
- [ ] Test 13: DynamoDB encryption at rest (KMS verified)

### Logging & Audit (5 tests)
- [ ] Test 14: CloudTrail captures login events
- [ ] Test 15: CloudWatch Logs do NOT contain PHI
- [ ] Test 16: API Gateway access logs sanitized
- [ ] Test 17: Audit log immutability (S3 Object Lock)
- [ ] Test 18: Log export to Glacier (tamper-evident)

### Resilience & DoS (4 tests)
- [ ] Test 19: WAF rate limiting blocks high-traffic IP
- [ ] Test 20: API Gateway throttling returns 429
- [ ] Test 21: Lambda concurrency limit enforced
- [ ] Test 22: DynamoDB auto-scaling under load

### Security Controls (4 tests)
- [ ] Test 23: IAM role least-privilege (deny Scan, allow GetItem only)
- [ ] Test 24: Dependency scanning identifies CVEs
- [ ] Test 25: SAST scan detects unsafe code patterns
- [ ] Test 26: TLS 1.2 minimum enforced (SSL/TLS audit)

### Compliance & Governance (4 tests)
- [ ] Test 27: No PHI in logs (search for sample data, confirm zero results)
- [ ] Test 28: CloudTrail enabled for all resources
- [ ] Test 29: KMS encryption key policy reviewed (no excessive access)
- [ ] Test 30: Quarterly CVE/vulnerability scan scheduled

---

**Last Updated:** 2026-03-14