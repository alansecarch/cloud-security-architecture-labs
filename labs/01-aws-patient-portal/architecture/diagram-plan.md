# Architecture Diagram Plan – Lab 01: AWS Patient Portal

## Overview

This document defines the nodes, edges, trust boundaries, and security annotations for the Lab 01 patient portal architecture diagram. The diagram covers the full request path from end-user browser to data store, reflecting the CloudFront + WAF → API Gateway → Lambda → DynamoDB architecture described in the README.

---

## Nodes

| Node | Type | Description |
|------|------|-------------|
| **Browser (Patient)** | External actor | Patient end-user accessing the portal via HTTPS |
| **Browser (Staff)** | External actor | Staff member accessing administrative endpoints via HTTPS |
| **Amazon CloudFront** | AWS service | CDN distribution; TLS termination, HTTPS enforcement, caching |
| **AWS WAF** | AWS service | Web ACL attached to CloudFront; rate-based rules, OWASP managed rule groups |
| **Amazon API Gateway** | AWS service | REST API; Cognito JWT authorizer, request validation, throttling, usage plans |
| **Amazon Cognito User Pool** | AWS service | Identity provider; user registration, login, MFA enforcement, JWT issuance |
| **Lambda – GetPatientRecord** | AWS service | Read-only Lambda; retrieves caller's own patient record |
| **Lambda – UpdatePatientRecord** | AWS service | Write Lambda (staff only); updates a patient record |
| **Lambda – AdminListPatients** | AWS service | Admin Lambda (staff only); lists all patients via GSI |
| **Amazon DynamoDB – Patients** | AWS service | Primary data store for patient records; partition key = user ID |
| **AWS KMS – PatientsCMK** | AWS service | Customer-managed key for DynamoDB encryption at rest |
| **AWS CloudTrail** | AWS service | Records all AWS API calls across account |
| **Amazon CloudWatch Logs** | AWS service | Receives Lambda structured logs; metric filters and alarms |
| **CloudTrail S3 Bucket** | AWS service | Immutable storage for CloudTrail log files; versioning + KMS encryption |

---

## Edges

| Source | Destination | Protocol / Auth | Notes |
|--------|-------------|----------------|-------|
| Browser (Patient/Staff) | CloudFront | HTTPS (TLS 1.2+) | All external traffic enters via CloudFront |
| CloudFront | WAF | Internal (same service) | WAF web ACL evaluates every request before forwarding |
| WAF | API Gateway | HTTPS | WAF forwards allowed requests to API Gateway origin |
| API Gateway | Cognito User Pool | HTTPS (JWKS fetch) | Authorizer validates JWT signature and claims |
| API Gateway | Lambda – GetPatientRecord | AWS internal (IAM) | Invoked for `GET /records/{id}` routes |
| API Gateway | Lambda – UpdatePatientRecord | AWS internal (IAM) | Invoked for `PUT /records/{id}` routes (staff only) |
| API Gateway | Lambda – AdminListPatients | AWS internal (IAM) | Invoked for `GET /admin/patients` routes (staff only) |
| Lambda – GetPatientRecord | DynamoDB – Patients | AWS internal (IAM role) | `dynamodb:GetItem` scoped to caller's partition key |
| Lambda – UpdatePatientRecord | DynamoDB – Patients | AWS internal (IAM role) | `dynamodb:UpdateItem` scoped to target partition key |
| Lambda – AdminListPatients | DynamoDB – Patients | AWS internal (IAM role) | `dynamodb:Query` on staff-access GSI |
| DynamoDB – Patients | KMS – PatientsCMK | AWS internal | Encryption/decryption of data at rest |
| Lambda (all) | CloudWatch Logs | AWS internal | Structured log output (no PHI) |
| AWS services (all) | CloudTrail | AWS internal | Automatic API call recording |
| CloudTrail | CloudTrail S3 Bucket | AWS internal (HTTPS) | Log file delivery with integrity validation |
| CloudWatch Logs | CloudWatch Alarms | AWS internal | Metric filters trigger alarms on anomalous patterns |

---

## Trust Boundaries

| Boundary | Description | What Crosses It |
|----------|-------------|-----------------|
| **Internet / AWS Edge** | Separates the public internet from CloudFront + WAF | All HTTPS requests from browsers; responses back to browsers |
| **AWS Edge / API Layer** | Separates CloudFront+WAF from API Gateway and Cognito | Authenticated, WAF-validated HTTPS requests |
| **API Layer / Compute Layer** | Separates API Gateway from Lambda functions | IAM-invoked Lambda calls carrying validated JWT context |
| **Compute Layer / Data Layer** | Separates Lambda from DynamoDB and KMS | IAM-authorized DynamoDB API calls; KMS decrypt calls |
| **Data Layer / Audit Layer** | Separates production data services from logging infrastructure | CloudTrail events, CloudWatch log streams, S3 log delivery |

---

## Security Annotations

- **TLS 1.2+ enforced at edge:** CloudFront security policy `TLSv1.2_2021`; all HTTP traffic is blocked or redirected.
- **HSTS on all responses:** `Strict-Transport-Security: max-age=31536000; includeSubDomains` header set by CloudFront.
- **WAF rate-based rule:** Source IPs exceeding 2,000 requests in 5 minutes are blocked for the duration of the window.
- **JWT validation (iss + aud + exp):** API Gateway Cognito authorizer enforces all three claims on every request; 15-minute access token TTL.
- **Staff-only MFA:** Cognito User Pool policy mandates TOTP MFA for the `staff` group; no exceptions.
- **Per-function IAM roles:** Each Lambda function uses a dedicated execution role with minimum required actions; explicit `Deny` for `dynamodb:Scan` on read-only roles.
- **DynamoDB CMK encryption:** `PatientsCMK` is a customer-managed KMS key; key usage is logged in CloudTrail; key rotation enabled annually.
- **Immutable audit trail:** CloudTrail log file integrity validation is enabled; CloudTrail S3 bucket denies `s3:DeleteObject` to all non-audit principals.
- **No PHI in logs:** Lambda logging policy prohibits payload logging; API Gateway access log format excludes body content.
- **Lambda reserved concurrency:** Each function has a configured reserved concurrency cap to prevent account-level concurrency exhaustion.
