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