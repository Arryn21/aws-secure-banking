# Threat Model — AWS Secure Banking Architecture

**Methodology:** STRIDE (Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege)

---

## Assets Being Protected

| Asset | Classification | Value |
|-------|---------------|-------|
| Customer PII (name, email, DOB) | Confidential | High |
| Account balances | Confidential | Critical |
| Transaction history | Confidential | High |
| Authentication credentials | Secret | Critical |
| Audit logs | Compliance | High |
| AWS API credentials | Secret | Critical |

---

## Threat Scenarios & Mitigations

### T1 — Credential Compromise (STRIDE: Spoofing)
**Scenario:** Attacker obtains AWS IAM access keys or a Cognito user's password.

**Attack path:**
1. Developer accidentally commits IAM access keys to GitHub
2. Attacker scrapes GitHub, finds keys
3. Uses keys to exfiltrate DynamoDB table, pivot to other services

**Mitigations:**
- No long-lived IAM access keys for human users (Cognito handles user auth)
- Lambda uses execution roles (no keys, rotated automatically)
- Secrets Manager stores DB credentials — never in code or env vars
- GuardDuty detects anomalous API calls from new IP/region
- CloudTrail alarm fires on any IAM key creation event

**Residual risk:** Low. Service-to-service auth uses IAM roles, not keys.

---

### T2 — SQL Injection via Transaction Endpoint (STRIDE: Tampering)
**Scenario:** Attacker sends crafted payload to `/transactions` endpoint to extract all account data.

**Attack path:**
1. POST `/transactions` with `userId = "' OR '1'='1"`
2. Unsanitized query executes against RDS, returns all rows
3. Attacker dumps entire accounts table

**Mitigations:**
- AWS WAF managed rule (AWSManagedRulesSQLiRuleSet) blocks at API Gateway
- Lambda uses parameterized queries — even if WAF is bypassed, injection fails
- IAM database authentication — even with credentials, access is scoped to specific Lambda role

**Residual risk:** Very Low. Three independent controls must all fail simultaneously.

---

### T3 — Unauthorized Data Access via Misconfigured S3 (STRIDE: Information Disclosure)
**Scenario:** Audit logs S3 bucket accidentally made public during configuration change.

**Attack path:**
1. Admin runs `aws s3api put-bucket-acl --acl public-read`
2. Attacker discovers bucket via S3 bucket enumeration tools
3. Downloads 6 months of CloudTrail logs, learns API usage patterns and resource names

**Mitigations:**
- S3 Block Public Access enabled at account level AND bucket level (two layers)
- AWS Config rule `S3_BUCKET_LEVEL_PUBLIC_ACCESS_PROHIBITED` detects and alerts within minutes
- Object Lock prevents deletion of evidence even if compromised

**Residual risk:** Very Low. Config rule will flag within 10 minutes of any public access attempt.

---

### T4 — Privilege Escalation via IAM Misconfiguration (STRIDE: Elevation of Privilege)
**Scenario:** Lambda function's IAM role is over-permissioned. Attacker achieves RCE in Lambda and uses the role to assume other roles or access services outside scope.

**Attack path:**
1. SSRF vulnerability in Lambda allows reading `169.254.169.254` (IMDS)
2. Retrieves Lambda execution role credentials
3. Uses credentials to call `iam:CreateRole`, `iam:AttachRolePolicy` — escalates to admin

**Mitigations:**
- Lambda role has no `iam:*` permissions — cannot create or modify roles
- Lambda role scoped to specific DynamoDB table + Secrets Manager secret (no wildcards)
- IAM Access Analyzer used during development to verify least privilege
- CloudWatch alarm fires on any IAM policy change

**Residual risk:** Low. Even with SSRF, the compromised role cannot pivot to other services.

---

### T5 — DDoS / Credential Stuffing (STRIDE: Denial of Service)
**Scenario:** Bot network sends thousands of login attempts against Cognito user pool.

**Attack path:**
1. Attacker runs credential stuffing tool with leaked password database
2. 10,000 requests/minute to Cognito `/login` endpoint
3. Legitimate users locked out; Lambda invocations spike; costs increase

**Mitigations:**
- API Gateway usage plan: 100 req/min per IP
- Cognito Advanced Security Mode: detects compromised credentials, adaptive authentication
- WAF rate-based rule: block IPs exceeding 300 req/5min
- CloudWatch alarm on Lambda invocation spike triggers SNS alert

**Residual risk:** Medium. Distributed botnets with many IPs can still exceed rate limits. Future control: CAPTCHA on Cognito hosted UI.

---

### T6 — Insider Threat / Audit Log Tampering (STRIDE: Repudiation)
**Scenario:** Malicious insider deletes CloudTrail logs to cover unauthorized access to customer accounts.

**Attack path:**
1. Admin with S3 write access deletes objects in audit bucket
2. Compliance review finds no evidence of access — incident undetectable

**Mitigations:**
- S3 Object Lock (COMPLIANCE mode, 365 days) — even account root cannot delete locked objects
- CloudTrail log file integrity validation — tampered or deleted logs are detectable via hash chain
- Separate AWS account for audit bucket (not implemented in this demo, recommended for production)

**Residual risk:** Low for external attackers. For insider threat with account root: Object Lock + hash validation makes tampering detectable but not preventable without multi-account architecture.

---

## Controls Coverage Map

| Threat | Prevent | Detect | Respond |
|--------|---------|--------|---------|
| Credential Compromise | Secrets Manager, IAM roles | GuardDuty, CloudTrail | SNS alert, key revocation |
| SQL Injection | WAF, parameterized queries | WAF logs, RDS audit logs | Block at WAF |
| S3 Data Exposure | Block Public Access, bucket policy | AWS Config rule | Config remediation |
| Privilege Escalation | Least-privilege IAM | CloudTrail IAM alarm | SNS alert, role review |
| DDoS / Stuffing | Rate limiting, WAF | CloudWatch spike alarm | WAF IP block |
| Audit Tampering | Object Lock | Log integrity validation | Forensic hash check |
