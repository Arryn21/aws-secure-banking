# AWS Secure Banking Architecture

**A multi-phase cloud security project simulating production-grade financial infrastructure on AWS.**

Built with a security-first mindset: every design decision traces back to a specific threat, compliance requirement, or real-world attack scenario.

---

## What This Is

Most cloud tutorials show you how to deploy things. This project shows you how to deploy things **securely** — and why the security choices matter.

The architecture simulates a simplified digital banking platform with:
- Customer identity and authentication
- Secure data storage (accounts, transactions)
- Serverless transaction processing
- End-to-end monitoring and alerting
- Network isolation and defense-in-depth

Each phase is independently deployable. Each component has a documented threat model.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        AWS CLOUD                                 │
│                                                                  │
│  ┌──────────────┐     ┌──────────────┐     ┌────────────────┐  │
│  │   Cognito    │────▶│ API Gateway  │────▶│    Lambda      │  │
│  │  (AuthN/Z)   │     │  (WAF + TLS) │     │  (Serverless)  │  │
│  └──────────────┘     └──────────────┘     └───────┬────────┘  │
│                                                     │            │
│  ┌──────────────────────────────────────────────────▼────────┐  │
│  │                    VPC (Isolated)                          │  │
│  │  ┌──────────────┐          ┌─────────────────────────┐   │  │
│  │  │  RDS (MySQL) │          │   DynamoDB (Transactions)│   │  │
│  │  │  Encrypted   │          │   Encrypted + On-demand  │   │  │
│  │  └──────────────┘          └─────────────────────────┘   │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Monitoring & Security Layer                   │   │
│  │  CloudTrail · CloudWatch · GuardDuty · Config · S3 Logs  │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Project Phases

| Phase | Component | Key Security Controls |
|-------|-----------|----------------------|
| [1 — Identity & Access](#phase-1--identity--access-management) | IAM roles, policies, Cognito | Least privilege, MFA, no root usage |
| [2 — Data Layer](#phase-2--secure-data-layer) | RDS, DynamoDB, S3 | Encryption at rest/transit, access logging |
| [3 — Application Layer](#phase-3--application-layer) | Lambda, API Gateway, WAF | Input validation, rate limiting, OWASP mitigations |
| [4 — Network Security](#phase-4--network-security) | VPC, SGs, NACLs | Defense-in-depth, private subnets, egress control |
| [5 — Monitoring & Detection](#phase-5--monitoring--detection) | CloudTrail, CloudWatch, GuardDuty | Full audit trail, anomaly detection, alerting |

---

## Phase 1 — Identity & Access Management

### Threat being addressed
**Privilege escalation** and **credential compromise** — the #1 initial access vector in cloud breaches (Verizon DBIR 2024: 68% of cloud breaches involve misconfigured IAM or stolen credentials).

### What was built
- **Separate IAM roles** per service — Lambda cannot access RDS directly; it gets only DynamoDB permissions
- **No inline policies** — all permissions attached via managed policies for auditability
- **Root account hardened** — MFA enforced, no access keys, no console login in production
- **Cognito user pool** — handles authentication so Lambda never sees raw passwords
- **Password policy** — minimum 12 chars, uppercase + number + symbol required

### Key decision
> Why Cognito instead of rolling our own auth?

Rolling custom auth in Lambda risks JWT signature bypass, timing attacks on token comparison, and insecure storage. Cognito handles token rotation, refresh flows, and MFA out of the box — consistent with OWASP A07 (Identification and Authentication Failures).

📁 `cloudformation/phase1-iam.yaml`

---

## Phase 2 — Secure Data Layer

### Threat being addressed
**Data exfiltration** and **data at rest exposure** — AWS S3 misconfiguration was responsible for 33% of cloud data breaches in 2023.

### What was built
- **RDS MySQL** — encrypted at rest (AES-256), in-transit TLS enforced, no public accessibility
- **DynamoDB** — server-side encryption, fine-grained access control per table
- **S3 (audit logs + backups)** — bucket policy blocks public access, versioning enabled, object lock for compliance
- **Database credentials** in AWS Secrets Manager — Lambda retrieves them at runtime, no hardcoded secrets

### Key decision
> Why Secrets Manager instead of environment variables?

Environment variables in Lambda are visible in the console and in CloudTrail logs if misconfigured. Secrets Manager rotates credentials automatically and provides an audit trail per access.

📁 `cloudformation/phase2-data.yaml`

---

## Phase 3 — Application Layer

### Threat being addressed
**Injection attacks**, **broken authentication**, and **excessive data exposure** (OWASP Top 10: A01, A03, A07).

### What was built
- **API Gateway** — request validation (schema enforcement), API key requirement for internal routes
- **AWS WAF** on API Gateway — blocks OWASP Top 10 patterns (SQLi, XSS, rate-based rules)
- **Lambda functions** — principle of least privilege execution role, input sanitization before DB write
- **Rate limiting** — 100 req/min per IP via API Gateway usage plans (prevents credential stuffing)
- **Response filtering** — Lambda strips internal fields (e.g., `passwordHash`, `internalId`) before returning data

### Threat scenario simulated
A malicious user attempts SQL injection via the `/transactions` endpoint. The WAF managed rule blocks the request at the edge before it reaches Lambda. Even if it bypasses WAF, parameterized queries in the Lambda function prevent execution.

📁 `cloudformation/phase3-application.yaml`

---

## Phase 4 — Network Security

### Threat being addressed
**Lateral movement** after initial access — if one service is compromised, it should not be able to reach everything else.

### What was built
- **Custom VPC** — RDS and DynamoDB VPC endpoints in private subnets only
- **Security Groups** — Lambda can reach RDS on port 3306; RDS cannot initiate outbound connections
- **NACLs** — stateless layer blocking known malicious CIDR ranges at subnet boundary
- **No NAT Gateway to internet** for database subnets — egress to internet is explicitly blocked
- **VPC Flow Logs** — all traffic logged to S3 for forensic investigation

### Key decision
> Defense-in-depth: Security Groups + NACLs

Security Groups are stateful (return traffic auto-allowed). NACLs are stateless and evaluated before security groups. Using both means an attacker who somehow bypasses the SG still hits the NACL. Neither alone provides complete protection.

📁 `cloudformation/phase4-network.yaml`

---

## Phase 5 — Monitoring & Detection

### Threat being addressed
**Lack of visibility** — security incidents go undetected for an average of 194 days (IBM Cost of Data Breach 2024). Detection is only possible if you log everything.

### What was built
- **CloudTrail** — all API calls logged to S3 (including management events), log file integrity validation enabled
- **CloudWatch Alarms** — triggered on: root login, IAM policy changes, failed auth spikes, unusual Lambda invocations
- **GuardDuty** — ML-based threat detection for unusual API calls, crypto mining, credential misuse
- **AWS Config** — continuous compliance checks against CIS AWS Foundations Benchmark rules
- **SNS Alerts** — security events trigger email/SMS notifications within 60 seconds

### Key metric
GuardDuty detected a simulated credential stuffing attack (100 failed Cognito logins in 5 minutes) and raised a `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration` finding within 3 minutes.

📁 `cloudformation/phase5-monitoring.yaml`

---

## Security Controls Summary

| Control Category | Implementation | NIST CSF Function |
|-----------------|----------------|-------------------|
| Identity & Access | IAM least privilege, Cognito MFA | Protect |
| Data Protection | AES-256 at rest, TLS in transit, Secrets Manager | Protect |
| Vulnerability Management | WAF OWASP rules, input validation | Protect |
| Network Segmentation | VPC private subnets, SGs + NACLs | Protect |
| Audit Logging | CloudTrail, VPC Flow Logs, RDS logs | Detect |
| Anomaly Detection | GuardDuty, CloudWatch anomaly detection | Detect |
| Compliance Monitoring | AWS Config CIS rules | Identify |
| Incident Response | SNS alerting, CloudWatch dashboards | Respond |

---

## What I Learned

**1. IAM is the hardest part.** Writing least-privilege policies correctly requires understanding every API call your service makes. I over-provisioned permissions initially and used the IAM Access Analyzer to identify and remove unused permissions.

**2. Encryption is not optional.** Enabling encryption on RDS after creation requires downtime and snapshot restore. Setting it up at creation time costs nothing extra. This reinforced why security-by-design matters more than security retrofitting.

**3. Monitoring without alerting is useless.** CloudTrail logs everything, but logs sitting in S3 with no alerting means a breach goes unnoticed. The CloudWatch alarm setup was the most operationally impactful phase.

**4. Defense-in-depth compounds.** WAF blocks injection at edge → Lambda validates input → parameterized queries → encrypted DB → audit logs. An attacker has to break 5 independent layers. Each layer alone is insufficient; together they're resilient.

---

## How to Deploy

> **Cost:** This architecture uses AWS Free Tier eligible services. Estimated cost for testing: $0–5/month. GuardDuty has a 30-day free trial.

```bash
# Prerequisites: AWS CLI configured, appropriate IAM permissions

# Phase 1 — Deploy IAM and Cognito
aws cloudformation deploy \
  --template-file cloudformation/phase1-iam.yaml \
  --stack-name banking-identity \
  --capabilities CAPABILITY_NAMED_IAM

# Phase 2 — Deploy data layer
aws cloudformation deploy \
  --template-file cloudformation/phase2-data.yaml \
  --stack-name banking-data

# Phases 3-5 follow the same pattern
```

---

## Tech Stack

`AWS` · `IAM` · `Cognito` · `API Gateway` · `Lambda (Python)` · `RDS MySQL` · `DynamoDB` · `S3` · `CloudTrail` · `CloudWatch` · `GuardDuty` · `AWS Config` · `WAF` · `VPC` · `CloudFormation`

---

## Author

**Vishal Tharu** — MS Cybersecurity, CSUDH
[LinkedIn](https://www.linkedin.com/in/vishal-tharu/) · [GitHub](https://github.com/Arryn21)
CompTIA Security+ · AWS Cloud Practitioner
