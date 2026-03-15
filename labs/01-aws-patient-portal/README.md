# Lab 01 — AWS Patient Portal (Serverless) | Security Architecture

## Goal

Design a secure, small "patient portal" on AWS and document the security architecture decisions with a threat model (STRIDE). Primary driver: confidentiality of PHI.

## Scope (MVP)

- **Authentication:** Amazon Cognito
- **Edge:** CloudFront + AWS WAF
- **API:** API Gateway with Cognito Authorizer
- **Compute:** AWS Lambda
- **Data:** DynamoDB (encrypted with KMS)
- **Logging/Monitoring:** CloudTrail + CloudWatch (avoid logging PHI)

## Users

- **Patients:** No MFA required (MVP)
- **Staff:** MFA required

## High-Level Architecture

**Request flow (conceptual):**

```
User (Browser)
  │  HTTPS
  ▼
CloudFront (CDN + TLS termination)
  │
AWS WAF (rate limiting, rule-based filtering)
  │
API Gateway (HTTPS only, Cognito Authorizer)
  │  Bearer token validated
  ▼
AWS Lambda (business logic, authorization checks)
  │
Amazon DynamoDB (KMS-encrypted at rest)
```

**Logging side-channel (all layers):**
CloudTrail → CloudWatch Logs (PHI-free) → S3 (encrypted, Object Lock)

## Security Architecture Decisions

| Decision | Rationale |
|----------|-----------|
| Cognito for AuthN | Managed IdP; built-in MFA, token signing (RS256), and Cognito groups for RBAC |
| CloudFront + WAF at edge | DDoS mitigation and rate limiting before requests reach the API layer |
| API Gateway Cognito Authorizer | Token validation at the edge; Lambda only runs for authenticated requests |
| DynamoDB + KMS | PHI encrypted at rest; KMS key policy controls who can decrypt |
| CloudTrail + CloudWatch (no PHI in logs) | Non-repudiation; avoids PHI leakage into log infrastructure |
| Lambda least-privilege IAM | Limits blast radius if a Lambda function is compromised |

## Lab Artifacts

- [`threat-model.md`](./threat-model.md) — STRIDE analysis (10 threats)
- [`controls-mapping.md`](./controls-mapping.md) — Threat-to-control mapping with 30 validation tests
- [`architecture/diagram-plan.md`](./architecture/diagram-plan.md) — Architecture diagram nodes, edges, and boundaries

## Next Steps

1. Review the [threat model](./threat-model.md) to understand the STRIDE threats identified for this architecture.
2. Walk through the [controls mapping](./controls-mapping.md) to see how each threat is mitigated.
3. Use the validation checklists in the controls mapping to verify controls in a deployed environment.