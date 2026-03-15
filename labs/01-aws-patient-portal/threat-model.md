# STRIDE Threat Model

## Overview
This threat model outlines 10 potential threats to the AWS Patient Portal using the STRIDE framework.

### 1. Spoofing
**Description**: Attackers may impersonate legitimate users to gain unauthorized access.

### 2. Tampering
**Description**: An attacker might manipulate sensitive data, such as altering patient records.

### 3. Repudiation
**Description**: Users may deny actions taken within the portal if adequate logging isn't implemented.

### 4. Information Disclosure
**Description**: Sensitive patient information can be exposed due to inadequate data protection or insecure APIs.

### 5. Denial of Service
**Description**: An attacker may flood the portal with requests, leading to service downtime.

### 6. Elevation of Privilege
**Description**: A user may exploit vulnerabilities to gain higher access rights than intended.

### 7. SQL Injection
**Description**: Attackers may inject malicious SQL queries to manipulate the database.

### 8. Cross-Site Scripting (XSS)
**Description**: Malicious scripts may be injected into web pages viewed by other users, potentially stealing session tokens.

### 9. Phishing
**Description**: Attackers may trick users into providing login credentials through deceptive emails or websites.

### 10. Insecure APIs
**Description**: APIs that lack proper security measures may expose backend systems to unauthorized access.