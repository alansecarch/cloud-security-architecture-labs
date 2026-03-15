# Controls Mapping – Lab 01: AWS Patient Portal

This document maps each threat from the STRIDE threat model (T1–T10) to its specific security controls, with bullets per threat.

---

## Threat 1 – Staff Credential Theft (Spoofing)

- Cognito User Pool enforces TOTP MFA for all users in the `staff` group; authentication fails if MFA is not completed.
- Password policy: minimum 12 characters with complexity requirements (uppercase, lowercase, numbers, symbols).
- Account temporarily locked after 5 consecutive failed authentication attempts.
- CloudTrail records every Cognito authentication event (success and failure) with timestamp, user identity, and source IP.
- CloudWatch metric filter alerts on 3+ consecutive failures for the same username within a 5-minute window.
- Session access token TTL: 15 minutes; refresh token rotated on each use and expires after 8 hours of inactivity.

---

## Threat 2 – JWT Token Forgery (Spoofing)

- API Gateway Cognito authorizer validates the RS256 signature of every incoming JWT against the Cognito User Pool's public JWKS endpoint.
- Authorizer enforces `iss` claim: token must originate from the configured User Pool ARN.
- Authorizer enforces `aud` claim: token must target the correct app client ID.
- Authorizer enforces `exp` claim: tokens past their expiry are rejected with 401.
- Lambda functions independently verify the `cognito:groups` claim to enforce role-based access before any business logic executes.
- Token refresh rotation: each call to the Cognito refresh endpoint issues a new refresh token and invalidates the previous one.

---

## Threat 3 – Patient Record Modification (Tampering)

- Lambda resolves the target DynamoDB record from the caller's Cognito `sub` claim extracted from the validated JWT, not from user-supplied URL parameters.
- IAM condition `dynamodb:LeadingKeys` on the Lambda execution role restricts writes to partition keys matching the caller's `sub`.
- API Gateway JSON schema validation rejects requests with unexpected fields or malformed bodies before Lambda is invoked.
- Lambda performs secondary input validation (field type, length, allowed values) using strict schema checks before constructing DynamoDB expressions.
- All DynamoDB writes use parameterized `UpdateExpression` (boto3 `Key()` and `Attr()` objects); no string concatenation in query construction.

---

## Threat 4 – Request Interception and Modification / MITM (Tampering)

- CloudFront security policy set to `TLSv1.2_2021`, disabling TLS 1.0, TLS 1.1, and all SSL versions.
- API Gateway endpoint is HTTPS-only; HTTP requests are not accepted.
- CloudFront origin protocol policy enforces HTTPS between CloudFront and API Gateway.
- `Strict-Transport-Security: max-age=31536000; includeSubDomains` header added to all CloudFront responses.
- AWS WAF blocks requests that arrive without a valid HTTPS handshake or that match Known Bad Inputs managed rule group.
- API Gateway request validation rejects payloads that do not conform to defined JSON schemas.

---

## Threat 5 – Denial of Administrative Action (Repudiation)

- AWS CloudTrail is enabled in all regions with log file integrity validation; trail logs are delivered to a dedicated S3 bucket.
- CloudTrail captures all DynamoDB API calls (GetItem, UpdateItem, DeleteItem, Query) with full user identity context.
- Lambda writes a structured audit log entry to CloudWatch Logs for every mutating action: action type, user `sub`, resource ID, timestamp, and outcome.
- CloudTrail S3 bucket has versioning enabled, server-side encryption (KMS), and a bucket policy that denies `s3:DeleteObject` to all principals except the designated audit role.
- CloudTrail log file integrity validation uses SHA-256 hashing; any deletion or modification of a log file is detectable.
- CloudWatch Logs log group retention is set to 365 days; logs are exported to S3 Glacier after 90 days for long-term immutable storage.

---

## Threat 6 – PHI Leakage via Application Logs (Information Disclosure)

- Lambda logging policy: functions log only non-PHI metadata — action type, Cognito `sub`, resource ID, HTTP status code, and latency. Request payloads and response bodies are never logged.
- API Gateway access logs are configured to record method, resource path, status code, latency, and request ID only; body content is excluded from the log format string.
- X-Ray active tracing is disabled for Lambda functions that process patient record data; only anonymous performance metrics are emitted.
- CloudWatch Logs log group access is restricted to the `security-audit` IAM role via a resource-based policy; developer roles are denied `logs:GetLogEvents` in production.
- KMS encryption is applied to the CloudTrail S3 bucket and CloudWatch Logs log group using a customer-managed CMK with key usage logged in CloudTrail.
- Quarterly CloudWatch Logs Insights queries search for common PHI patterns (date-of-birth formats, MRN patterns) to detect accidental leakage.

---

## Threat 7 – Unauthenticated Access to Patient Data (Information Disclosure)

- API Gateway Cognito authorizer is applied to every route; no route is configured without an authorizer.
- The authorizer uses a `deny` effect by default; access is granted only when the JWT is valid, unexpired, and from the configured User Pool.
- API Gateway resource policy additionally restricts access to requests originating via CloudFront (source IP range of CloudFront edge nodes), preventing direct invocation of the API Gateway URL.
- DynamoDB tables are not publicly accessible; access requires valid AWS credentials scoped to the Lambda execution roles.
- All DynamoDB data is encrypted at rest with a customer-managed KMS CMK; decryption requires an explicit `kms:Decrypt` permission in the key policy.

---

## Threat 8 – API Flood / Resource Exhaustion (Denial of Service)

- AWS WAF rate-based rule blocks any source IP that sends more than 2,000 requests within a 5-minute rolling window; blocked IPs are automatically released after the window resets.
- AWS Managed Rule Groups (Core Rule Set, Known Bad Inputs) are enabled on the WAF web ACL to block malformed and attack-pattern traffic before it reaches the origin.
- API Gateway usage plan sets burst limit to 10,000 requests/second and steady-state limit to 5,000 requests/second at the account level.
- Per-method throttling applies tighter limits to write endpoints (e.g., `POST /records`: 100 req/s, `PUT /records/{id}`: 50 req/s).
- Each Lambda function has a reserved concurrency limit to prevent a single function from consuming the full account concurrency pool.
- CloudWatch alarms trigger on WAF blocked request rate, API Gateway 429 error rate, and Lambda throttle count; alarms route to an operational runbook via CloudWatch.

---

## Threat 9 – Patient Escalates to Staff Role (Elevation of Privilege)

- Lambda authorization layer checks the `cognito:groups` claim extracted from the validated JWT and returns a 403 if the caller is not in the `staff` group, before any database access occurs.
- Cognito group membership is managed by administrators only; the `staff` group has `AdminAddUserToGroup` restricted to the `iam-admin` role.
- API Gateway resource policy on `/admin/*` routes requires the `staff` group to be present in the authorizer context.
- IAM execution role for patient-facing Lambda functions explicitly omits `dynamodb:Scan` and `dynamodb:Query` without partition key conditions, limiting data accessible even if authorization logic fails.
- DynamoDB `dynamodb:LeadingKeys` condition in patient Lambda IAM policies prevents cross-user record access at the IAM enforcement layer.
- Explicit security test cases (patient JWT → staff endpoint) are part of the CI pipeline; failure blocks deployment.

---

## Threat 10 – Overpermissioned Lambda Role (Elevation of Privilege)

- Each Lambda function is deployed with a unique, dedicated IAM execution role; no shared roles across functions.
- Each execution role grants only the specific DynamoDB actions required by that function on the specific table ARN (no wildcards on actions or resources).
- Read-only Lambda roles include an explicit `Deny` statement for `dynamodb:Scan`, `dynamodb:DeleteItem`, and `dynamodb:PutItem`.
- KMS key policy lists the exact Lambda execution role ARNs permitted to call `kms:Decrypt`; no wildcards.
- Lambda functions run in a VPC with no public subnet association; internet egress is controlled via a NAT gateway with an allowlist security group.
- IAM Access Analyzer is enabled; any policy that grants access to resources outside the account generates a finding that is reviewed weekly.
- Lambda function URLs are disabled on all functions; invocation is only possible through API Gateway.
