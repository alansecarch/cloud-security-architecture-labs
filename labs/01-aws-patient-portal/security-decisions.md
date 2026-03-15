# Security Decisions – Lab 01: AWS Patient Portal

This document records the key security design decisions made for the patient portal architecture, the alternatives considered, and the rationale for each choice.

---

## 1. Cognito Authorizer vs. Custom Lambda Authorizer

**Decision:** Use the native Amazon API Gateway Cognito User Pool authorizer.

**Alternatives considered:**
- Custom Lambda authorizer that calls Cognito to verify tokens
- Third-party identity provider with custom OIDC validation in Lambda

**Rationale:**
The native Cognito authorizer validates the JWT signature (RS256), issuer, audience, and expiry entirely within API Gateway before the request reaches Lambda. This removes token validation logic from application code, eliminates a class of implementation errors, and reduces Lambda invocation cost. A custom Lambda authorizer would be preferable only if the authorization rules required business logic (e.g., attribute-based access control) that cannot be expressed through Cognito groups alone. For this portal, group-based routing (staff vs. patient) is sufficient, making the native authorizer the right choice.

---

## 2. Staff-Only MFA

**Decision:** Enforce TOTP MFA for the `staff` Cognito user group; leave patient MFA optional at this stage.

**Rationale:**
Staff accounts have write access to patient records and administrative functions. A compromised staff credential has a significantly higher blast radius than a compromised patient credential. Mandating MFA for staff is a proportionate, low-friction control because staff log in from managed devices on a predictable schedule. Patient MFA is deferred to reduce onboarding friction while the portal is in early adoption; the threat model records this explicitly so it can be revisited. MFA is enforced at the Cognito user pool level, not in application code, which prevents bypasses through the API layer.

---

## 3. Token Validation – iss, aud, exp Claims and Short TTL

**Decision:** API Gateway Cognito authorizer validates `iss`, `aud`, and `exp` on every request. Access token TTL is set to 15 minutes.

**Rationale:**
- **`iss` (issuer):** Confirms the token originated from the correct Cognito User Pool. This prevents tokens issued by a different pool (e.g., attacker-controlled) from being accepted.
- **`aud` (audience):** Confirms the token was issued for this specific app client. This prevents token reuse across different applications sharing the same pool.
- **`exp` (expiry):** Rejects tokens whose validity window has passed. Combined with a 15-minute access token TTL, the exposure window for a stolen token is bounded. Refresh tokens are longer-lived (8 hours) but are validated server-side and rotated on each use.
- Lambda functions perform a secondary check of the `cognito:groups` claim to enforce role-based access (staff vs. patient), providing defense in depth beyond what the authorizer enforces.

---

## 4. Least-Privilege IAM per Lambda Function

**Decision:** Each Lambda function is assigned a dedicated IAM execution role scoped to the minimum actions on the minimum resources it requires.

**Rationale:**
A single shared Lambda role with broad permissions would mean that a vulnerability in any function could be exploited to access all DynamoDB tables or call unintended AWS APIs. By creating per-function roles, the blast radius of a compromised function is limited to the data it legitimately accesses. Concretely:

- `GetPatientRecord` Lambda: `dynamodb:GetItem` on the `Patients` table, partition key scoped to the caller's user ID via IAM condition.
- `UpdatePatientRecord` Lambda (staff only): `dynamodb:UpdateItem` on the `Patients` table, no `Scan` or `DeleteItem`.
- `AdminListPatients` Lambda (staff only): `dynamodb:Query` on the `Patients` table with a GSI, no cross-table access.

All roles explicitly deny `dynamodb:Scan` except where operationally required and reviewed.

---

## 5. WAF + API Gateway Throttling

**Decision:** Deploy AWS WAF in front of CloudFront with rate-based rules, and apply API Gateway usage plans with per-method throttling.

**Rationale:**
Two complementary layers of rate control are needed because they operate at different scopes:

- **WAF (edge, CloudFront):** Blocks IP-level abuse before traffic reaches the origin. Rate-based rules block IPs exceeding 2,000 requests per 5-minute window. AWS Managed Rule Groups (Core Rule Set, Known Bad Inputs) block common OWASP Top 10 attack patterns without custom signature authoring.
- **API Gateway throttling (per-method):** Limits burst (10,000 req/s) and steady-state (5,000 req/s) traffic at the API level, providing protection even if WAF is misconfigured or bypassed via a direct API URL. Per-method limits allow tighter controls on write endpoints (e.g., 100 req/s on `POST /records`).

Together, these layers prevent resource exhaustion of Lambda concurrency and DynamoDB capacity, and ensure predictable costs.

---

## 6. Logging Strategy – Avoiding PHI in Logs

**Decision:** Structured logging in Lambda logs only non-PHI metadata; API Gateway access logs exclude request/response bodies; CloudTrail provides the authoritative AWS API audit trail.

**Rationale:**
Logging PHI (patient names, dates of birth, diagnosis codes, medication names) in CloudWatch Logs or API Gateway access logs creates a secondary data store that is harder to control, classify, and protect than DynamoDB. If PHI appears in logs, it becomes subject to the same retention, access control, and encryption requirements as the primary data store—but without the same guardrails.

The logging strategy is therefore:

- **Lambda:** Log action type, user ID (Cognito `sub`), resource ID, timestamp, and outcome (success/error code). Do not log request payloads or response bodies.
- **API Gateway access logs:** Log method, path, status code, latency, and request ID. Exclude body content.
- **CloudTrail:** Provides an immutable record of all AWS API calls (DynamoDB operations, KMS key usage, IAM changes). CloudTrail logs are stored in a dedicated S3 bucket with server-side encryption and no public access.
- **CloudWatch alarms:** Alert on anomalous patterns (e.g., spike in 4xx errors, high Lambda error rate) without requiring PHI to be present in alarm evaluation.

Log access is restricted to the `security-audit` IAM role; developers do not have production log read access by default.
# Security Decisions for Lab 01

## Cognito Authorizer
- Utilize AWS Cognito to handle user authentication and authorization, enhancing security by managing user identities within the application.

## Staff-Only MFA
- Implement Multi-Factor Authentication (MFA) for staff access to sensitive operations, ensuring that access requires both a password and a second verification method.

## Token Validation
- Enforce strict validation of JWT tokens for API requests, ensuring that tokens are not expired and are issued by a trusted authority.

## Least-Privilege IAM
- Apply the principle of least privilege when configuring IAM roles and policies, granting only the permissions necessary for performing required tasks.

## Rate Limiting
- Establish rate limiting on APIs to prevent abuse and protect against DDoS attacks, ensuring fair usage and maintaining service availability.

## Logging Strategy
- Implement a comprehensive logging strategy that captures access logs, error logs, and audit trails. Ensure logs are stored securely and are accessible for monitoring and analysis without involving SNS or S3.
