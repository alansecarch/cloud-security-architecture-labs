Lab 01 — AWS Patient Portal (Serverless) | Security Architecture
Goal

Design a secure, small “patient portal” on AWS and document the security architecture decisions with a threat model (STRIDE). Primary driver: confidentiality of PHI.

Scope (MVP)
Authentication: Amazon Cognito
Edge: CloudFront + AWS WAF
API: API Gateway with Cognito Authorizer
Compute: AWS Lambda
Data: DynamoDB (encrypted with KMS)
Logging/Monitoring: CloudTrail + CloudWatch (avoid logging PHI)
Users
Patients (no MFA required for MVP)
Staff (MFA required)
High-Level Architecture

Request flow (conceptual):

User → CloudFront (HTTPS)
CloudFront → WAF → API Gateway
API Gateway (Cognito Authorizer) → Lambda
Lambda → DynamoDB

Diagram:

See: architecture/diagram-plan.md (PNG will be added later)
Key Security Decisions (summary)
Use API Gateway + Cognito Authorizer to reduce custom auth code.
Enforce staff-only MFA in Cognito to reduce high-impact account takeover risk.
Use least-privilege IAM roles per Lambda function.
Encrypt data at rest with KMS and enforce TLS in transit.
Log identity and security-relevant events; avoid logging PHI.
Primary Deliverable
Threat model: threat-model.md (STRIDE table)
Files in this Lab
threat-model.md — STRIDE threats + mitigations + validation notes
security-decisions.md — tradeoffs and rationale
controls-mapping.md — traceability from threats to controls
architecture/diagram-plan.md — AWS-icons diagram plan
Next Steps
Create an AWS-icons diagram and export as architecture.png (later).
Keep controls-mapping.md aligned to threat # references.
