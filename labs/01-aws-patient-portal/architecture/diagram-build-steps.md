# Diagram Build Guide: AWS Patient Portal Architecture (draw.io)

This guide provides step-by-step instructions for building the Lab 01 architecture diagram in **diagrams.net** (draw.io) using AWS icons.

---

## Canvas Setup

### Size & Orientation
- **Canvas Dimensions:** 1920 x 1200 pixels (landscape banner format)
- **Grid:** Enable snap-to-grid (20px intervals)
- **Zoom Level:** Start at 100%, adjust to 75% or 50% for overview
- **Background:** White or light gray for clarity

### File Properties
- **Format:** Export as PNG (1920 x 1200) for GitHub README embedding
- **Filename:** `architecture.png`
- **Include Border:** Yes (4px gray border)

---

## Step 1: Add External Boundary (Internet)

1. **Rectangle (Light Red/Orange)**
   - Size: Full width, ~150px height
   - Position: Top-left (0, 0), spans entire width
   - Label: "Internet / Untrusted Network"
   - Style: Dashed border, fill=`#FFE6E6`, no shadow

2. **Add Two Browser Icons (inside Internet boundary)**
   - **Search:** "Customer" or "user" in draw.io shapes
   - **Icon 1 (Patient):** Position left within boundary, label "Browser (Patient)"
   - **Icon 2 (Staff):** Position center-left, label "Browser (Staff)"
   - Size: ~60px each

---

## Step 2: Add Edge Boundary (CloudFront + WAF)

1. **Rectangle (Light Blue)**
   - Position: Below Internet boundary (~160px from top), same width
   - Height: ~150px
   - Label: "AWS Edge (DDoS Protection, Rate Limiting)"
   - Style: Solid border, fill=`#E6F2FF`, no shadow

2. **CloudFront Icon**
   - **AWS Icon:** Search "CloudFront" in draw.io AWS icon set
   - Position: Left side of edge boundary (~120px from left)
   - Size: ~80px
   - Label (below): "CloudFront"
   - Add callout box: "TLS 1.2+ Enforced"

3. **AWS WAF Icon**
   - **AWS Icon:** Search "WAF" or "Web Application Firewall"
   - Position: Right of CloudFront (~300px from left)
   - Size: ~80px
   - Label (below): "AWS WAF"
   - Add callout box: "Rate Limit: >2000 req/5min"

---

## Step 3: Add API Layer Boundary

1. **Rectangle (Light Yellow)**
   - Position: Below edge boundary (~330px from top)
   - Height: ~150px
   - Label: "AWS API Layer (Authentication)"
   - Style: Solid border, fill=`#FFFACD`, no shadow

2. **API Gateway Icon**
   - **AWS Icon:** Search "API Gateway"
   - Position: Left-center of API layer (~120px from left)
   - Size: ~80px
   - Label: "API Gateway"
   - Add callout box: "Request Validation\nJSON Schema"

3. **Cognito Icon**
   - **AWS Icon:** Search "Cognito" or "Identity Provider"
   - Position: Right of API Gateway (~300px from left)
   - Size: ~80px
   - Label: "Cognito\nUser Pool"
   - Add callout box: "JWT Validation\n(iss, aud, exp)\n1-hour TTL"

---

## Step 4: Add Compute Layer Boundary

1. **Rectangle (Light Orange)**
   - Position: Below API layer (~510px from top)
   - Height: ~150px
   - Label: "AWS Compute Layer (Least-Privilege IAM)"
   - Style: Solid border, fill=`#FFE6CC`, no shadow

2. **Lambda Icons (3 functions)**
   - **AWS Icon:** Search "Lambda"
   - Positions (left to right):
     - **Lambda 1 (GetPatientRecord):** ~80px from left, size ~70px
       - Label: "GetPatientRecord\n(Read-only)"
       - Callout: "IAM: GetItem only"
     - **Lambda 2 (UpdatePatientRecord):** ~280px from left
       - Label: "UpdatePatientRecord\n(Staff only)"
       - Callout: "IAM: UpdateItem only"
     - **Lambda 3 (AdminListPatients):** ~480px from left
       - Label: "AdminListPatients\n(Staff only)"
       - Callout: "IAM: Query on GSI"

---

## Step 5: Add Data Layer Boundary

1. **Rectangle (Light Green)**
   - Position: Below compute layer (~690px from top)
   - Height: ~120px
   - Label: "AWS Data Layer (Encryption at Rest)"
   - Style: Solid border, fill=`#E6F9E6`, no shadow

2. **DynamoDB Icon**
   - **AWS Icon:** Search "DynamoDB"
   - Position: Left-center (~120px from left)
   - Size: ~80px
   - Label: "DynamoDB\nPatients Table"
   - Callout: "Partition Key = User ID"

3. **KMS Icon**
   - **AWS Icon:** Search "KMS" or "Key Management"
   - Position: Right of DynamoDB (~300px from left)
   - Size: ~80px
   - Label: "KMS\nPatientsCMK"
   - Callout: "CMK Encryption\nKey Rotation: Annual"

---

## Step 6: Add Logging/Monitoring Boundary

1. **Rectangle (Light Gray)**
   - Position: Below data layer (~840px from top)
   - Height: ~120px
   - Label: "AWS Logging & Monitoring (Immutable Audit Trail)"
   - Style: Solid border, fill=`#F0F0F0`, no shadow

2. **CloudTrail Icon**
   - **AWS Icon:** Search "CloudTrail"
   - Position: Left-center (~80px from left)
   - Size: ~70px
   - Label: "CloudTrail"
   - Callout: "S3 Object Lock\nNo PHI"

3. **CloudWatch Logs Icon**
   - **AWS Icon:** Search "CloudWatch" or "Logs"
   - Position: Center (~280px from left)
   - Size: ~70px
   - Label: "CloudWatch\nLogs"
   - Callout: "Redaction Filters\nNo PHI"

4. **KMS Icon (for logs)**
   - **AWS Icon:** Search "KMS"
   - Position: Right (~480px from left)
   - Size: ~70px
   - Label: "KMS\n(Log Encryption)"

---

## Step 7: Add Arrows (Request Flow)

### From Browser to CloudFront
- **Arrow:** Solid green arrow from Browser → CloudFront
- **Label:** "HTTPS\n(TLS 1.2+)"
- **Style:** Weight=2px

### From CloudFront to WAF
- **Arrow:** Solid green arrow (curved)
- **Label:** "Request\nEvaluation"
- **Style:** Weight=2px

### From WAF to API Gateway
- **Arrow:** Solid green arrow (if allowed)
- **Label:** "HTTPS\n(Validated)"
- **Style:** Weight=2px

### From API Gateway to Cognito
- **Arrow:** Dashed blue arrow (bidirectional for JWT validation)
- **Label:** "JWT Token\nValidation"
- **Style:** Weight=1.5px, dashed

### From API Gateway to Lambda Functions
- **Arrow 1:** Solid arrow to GetPatientRecord
  - **Label:** "GET /records/{id}"
- **Arrow 2:** Solid arrow to UpdatePatientRecord
  - **Label:** "PUT /records/{id}\n(Staff only)"
- **Arrow 3:** Solid arrow to AdminListPatients
  - **Label:** "GET /admin/patients\n(Staff only)"
- **Style:** Weight=2px each

### From Lambda to DynamoDB
- **Arrow 1:** Solid purple arrow from GetPatientRecord → DynamoDB
  - **Label:** "GetItem\n(Partition Key)"
- **Arrow 2:** Solid purple arrow from UpdatePatientRecord → DynamoDB
  - **Label:** "UpdateItem\n(Scoped)"
- **Arrow 3:** Solid purple arrow from AdminListPatients → DynamoDB
  - **Label:** "Query\n(GSI)"
- **Style:** Weight=2px each

### From DynamoDB to KMS
- **Arrow:** Dashed red arrow (encryption/decryption)
- **Label:** "Encrypt/Decrypt\n(CMK)"
- **Style:** Weight=1.5px, dashed

### From Lambda to CloudWatch Logs
- **Arrow:** Dashed gray arrow from all Lambda functions (combine into one line)
- **Label:** "Structured Logs\n(no PHI)"
- **Style:** Weight=1.5px, dashed

### From CloudTrail to S3
- **Arrow:** Dashed gray arrow
- **Label:** "Immutable\nArchive"
- **Style:** Weight=1.5px, dashed

---

## Step 8: Add Security Callout Boxes

Add floating text boxes around the diagram:

1. **Top-Left (above CloudFront):**
   - Text: "⚠ TLS 1.2+ Only\nHTTP → Redirect"
   - Style: Yellow sticky note, font size 10px

2. **Top-Center (above WAF):**
   - Text: "⚠ WAF Web ACL\nOWASP Rules Active"
   - Style: Yellow sticky note, font size 10px

3. **API Layer (right side):**
   - Text: "🔐 JWT Required\n(iss + aud + exp)"
   - Style: Blue sticky note, font size 10px

4. **Compute Layer (right side):**
   - Text: "🔐 Least-Privilege IAM\nNo wildcard actions"
   - Style: Orange sticky note, font size 10px

5. **Data Layer (right side):**
   - Text: "🔐 CMK Encryption\nKey rotation enabled"
   - Style: Green sticky note, font size 10px

6. **Logging Boundary (right side):**
   - Text: "📋 Immutable Audit Trail\nObject Lock + Integrity Validation"
   - Style: Gray sticky note, font size 10px

---

## Step 9: Add Legend

Place a legend box in the bottom-right corner of the canvas:

| Symbol | Meaning |
|--------|---------|
| Solid green arrow | HTTPS / authenticated request flow |
| Dashed blue arrow | JWT token validation (bidirectional) |
| Solid purple arrow | DynamoDB API call (IAM-scoped) |
| Dashed red arrow | KMS encryption/decryption |
| Dashed gray arrow | Audit log delivery |
| Colored rectangle | Trust boundary / security zone |

---

## Step 10: Final Review Checklist

Before exporting, verify the following:

- [ ] All six trust boundary rectangles are present and labeled
- [ ] All AWS icons are from the official AWS icon set (not generic shapes)
- [ ] Every arrow has a label describing the protocol and auth mechanism
- [ ] All three Lambda functions have IAM callout boxes
- [ ] DynamoDB and KMS icons are both present in the data layer
- [ ] CloudTrail, CloudWatch Logs, and KMS are in the logging boundary
- [ ] Security callout boxes are visible and not overlapping icons
- [ ] Legend is present in the bottom-right corner
- [ ] Canvas is 1920 x 1200 pixels
- [ ] Exported PNG is named `architecture.png`

---

## Export Instructions

1. **File → Export As → PNG**
2. Set width to **1920** and height to **1200**
3. Check **"Include a copy of my diagram"** (embeds XML for future editing)
4. Check **"Transparent Background"**: No (use white)
5. Click **Export** and save as `architecture.png` in this directory
6. Commit the exported PNG alongside this guide
