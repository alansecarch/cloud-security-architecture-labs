# Lab 01 – AWS Patient Portal: Security Architecture

## Overview

This lab designs and evaluates the security architecture of a serverless patient portal built on AWS. The scope covers the full request path from browser to data store, with particular attention to access control, data protection, and audit logging for a HIPAA-adjacent workload.

## Architecture Components

| Layer | AWS Service | Security Role |
|-------|------------|---------------|
| Edge / CDN | Amazon CloudFront + AWS WAF | TLS termination, geo-blocking, rate limiting, OWASP managed rule groups |
| API | Amazon API Gateway (REST) | Cognito JWT authorizer, request throttling, input schema validation |
| Auth | Amazon Cognito User Pools | User registration/login, staff-only MFA enforcement, token issuance |
| Compute | AWS Lambda (per function) | Business logic, secondary JWT claim validation, least-privilege IAM roles |
| Data | Amazon DynamoDB + AWS KMS | Patient records, encryption at rest with customer-managed CMK |
| Audit | AWS CloudTrail + Amazon CloudWatch Logs | API-level audit trail, Lambda execution logs, metric-based alerting |

## Key Security Design Decisions

- **CloudFront + WAF at the edge** — all traffic enters through CloudFront, which enforces HTTPS (TLS 1.2+) and applies WAF rules before requests reach API Gateway.
- **API Gateway Cognito authorizer** — tokens are validated cryptographically (RS256) at the API layer without custom authorizer code.
- **Staff-only MFA** — Cognito user pool policy mandates TOTP MFA for the `staff` group; patients are not subject to MFA at this stage.
- **Lambda least-privilege IAM** — each Lambda function has its own execution role scoped to the minimum DynamoDB actions needed (e.g., `GetItem` only for read functions).
- **DynamoDB encryption with KMS CMK** — enables key rotation, key policy enforcement, and CloudTrail-level key usage visibility.
- **CloudTrail + CloudWatch** — all AWS API calls recorded; Lambda logs structured without PHI; CloudWatch alarms on anomalous patterns.

## What This Lab Does NOT Cover

- S3-based file storage (out of scope for this lab iteration)
- SNS-based notifications (out of scope for this lab iteration)
- Frontend SPA deployment (assumed pre-built static site delivered via CloudFront)

## Learning Objectives

1. Understand how a layered AWS defense-in-depth architecture works from edge to data.
2. Evaluate trade-offs of API Gateway native Cognito authorizer versus a custom Lambda authorizer.
3. Apply least-privilege IAM design across multiple Lambda functions.
4. Configure WAF and API Gateway throttling to protect against denial-of-service and abuse.
5. Design a logging strategy that supports audit requirements without capturing PHI.
6. Use the STRIDE framework to enumerate threats and map controls to mitigations.