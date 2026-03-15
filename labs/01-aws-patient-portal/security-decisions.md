Security Decisions — Lab 01 (AWS Serverless Patient Portal)

This document captures key security architecture decisions and the rationale behind them. It is intentionally configuration-agnostic (no hard-coded thresholds, TTLs, or quotas) and focuses on defensible design choices.

1) API Gateway Cognito Authorizer vs Custom Lambda Authorizer

Decision: Use an API Gateway Cognito Authorizer for authentication. Rationale:

Reduces custom authorization code and associated security risk.
Leverages a managed integration for JWT validation and standard auth flows.
Keeps the "auth boundary" explicit at the API layer. Tradeoff:
Less flexibility for highly custom policies. Mitigation:
Enforce fine-grained authorization in Lambda (resource ownership checks, role/group checks).
2) One Cognito User Pool with Groups (Patients vs Staff)

Decision: Use one Cognito User Pool and separate Patients vs Staff using groups/claims. Rationale:

Simpler to manage than multiple pools for an MVP.
Enables role-based access patterns (e.g., staff-only endpoints) using token claims. Tradeoff:
Requires careful group assignment and verification. Mitigation:
Validate group/role claims at the API/Lambda layers and test negative cases.
3) Staff-Only MFA (MVP)

Decision: Require MFA for staff accounts; do not require MFA for patients in the MVP. Rationale:

Staff accounts typically have broader access and higher impact if compromised.
Patient MFA can add significant friction and support burden early on. Tradeoff:
Patient accounts remain more susceptible to account takeover. Mitigation:
Strong password policy, monitoring for suspicious login patterns, and rate limiting on auth-related paths. MFA for patients remains a future hardening step.
4) Defense-in-Depth Authorization (Not Just Authentication)

Decision: Treat authentication (who you are) and authorization (what you can do) as separate controls. Rationale:

A valid token should not automatically imply access to any patient record.
Prevents common issues like IDOR (insecure direct object reference). Implementation intent:
Lambda enforces object-level authorization: patient can only access their own records; staff access is scoped to explicit job functions and approved workflows.
5) Least-Privilege IAM per Lambda Function

Decision: Use separate IAM roles per Lambda function with only required permissions. Rationale:

Limits blast radius if one function is misused or compromised.
Improves auditability and clarity of what each function can access. Tradeoff:
More IAM policies to manage. Mitigation:
Keep roles and policies small, named clearly, and reviewed as part of the lab artifacts.
6) Data Protection and Key Management

Decision: Encrypt sensitive data at rest using KMS-managed keys and enforce encryption in transit. Rationale:

Protects PHI against data exposure from storage-layer compromise and accidental access.
Establishes clear "data boundary" controls appropriate for healthcare scenarios. Tradeoff:
Key policy and access management adds complexity. Mitigation:
Explicitly define which roles/services can use keys; review access paths as part of the controls mapping.
7) Rate Limiting and Abuse Protection at Multiple Layers

Decision: Apply throttling/abuse controls at the edge and API layers. Rationale:

Reduces risk of volumetric abuse, credential stuffing, and resource exhaustion.
Provides earlier rejection of bad traffic before it reaches Lambda and the data layer. Tradeoff:
Risk of blocking legitimate users if mis-tuned. Mitigation:
Monitor for false positives and adjust rules based on observed traffic patterns (in real systems).
8) Logging Strategy: Security-Relevant Events Without PHI

Decision: Log security-relevant events while intentionally avoiding PHI in logs. Rationale:

Logs are necessary for detection, investigation, and accountability.
Logging PHI increases exposure risk and complicates handling requirements. Implementation intent:
Log event metadata (who/what/when/result) and identifiers that are not PHI.
Restrict access to logs via IAM and audit log access via CloudTrail.
9) Separate "Design Artifacts" From "Deployable Code"

Decision: Keep Lab 01 documentation-first (architecture + threat model + controls) before adding deployable IaC/code. Rationale:

Reduces the risk of publishing insecure defaults or accidental secrets.
Keeps the portfolio focused on security architecture thinking and tradeoffs. Future direction:
If deployable artifacts are added later, they will use placeholders, secret scanning, and secure-by-default patterns.

Last updated: 2026-03-15