# Threat Model

## Overview

Identification of potential security threats and corresponding mitigations for the Wallet Service.

---

## Threat Categories

### 1. Authentication & Authorization Attacks

| Threat | Impact | Mitigation |
|--------|--------|------------|
| **Token Theft** | Unauthorized access | Short-lived tokens (15 min), refresh token rotation |
| **Privilege Escalation** | Admin access | RBAC enforcement, method-level security |
| **Credential Stuffing** | Account takeover | Rate limiting, CAPTCHA, MFA |
| **JWT Replay** | Unauthorized transactions | Idempotency keys, short TTL |

---

### 2. Injection Attacks

| Threat | Impact | Mitigation |
|--------|--------|------------|
| **SQL Injection** | Data breach | Parameterized queries (JPA), input validation |
| **NoSQL Injection** | Cache poisoning | Redis command sanitization |
| **LDAP Injection** | Directory compromise | Escape special characters |
| **Command Injection** | Server compromise | No shell command execution |

---

### 3. Financial Fraud

| Threat | Impact | Mitigation |
|--------|--------|------------|
| **Double Spending** | Financial loss | Idempotency keys, optimistic locking |
| **Replay Attack** | Duplicate transactions | Unique constraints, TTL expiration |
| **Balance Manipulation** | Ledger corruption | Immutable ledger, double-entry verification |
| **Discount Abuse** | Revenue loss | Usage limits, per-user caps |

---

### 4. Denial of Service

| Threat | Impact | Mitigation |
|--------|--------|------------|
| **Rate Abuse** | Service degradation | Redis rate limiting (per user/IP) |
| **Resource Exhaustion** | System crash | Request size limits, connection pooling |
| **DDoS** | Complete outage | CloudFlare/AWS Shield, auto-scaling |

---

### 5. Data Breaches

| Threat | Impact | Mitigation |
|--------|--------|------------|
| **Database Exposure** | PII leak | Encryption at rest (TDE), access controls |
| **Log Leakage** | Sensitive data exposure | PII masking, secure log storage |
| **Backup Theft** | Historical data breach | Encrypted backups, secure S3 buckets |

---

## Attack Scenarios & Mitigations

### Scenario 1: Replay Attack on Payment
**Threat**: Attacker intercepts and replays payment request.

**Mitigation**:
1. Idempotency keys with TTL
2. Unique constraint on (key, user_id, endpoint)
3. JWT short expiry (15 min)
4. Correlation ID tracking

---

### Scenario 2: Man-in-the-Middle (MITM)
**Threat**: Attacker intercepts communication.

**Mitigation**:
1. TLS 1.3 enforced
2. Certificate pinning (mobile apps)
3. mTLS for B2B
4. HSTS headers

---

### Scenario 3: Insider Threat
**Threat**: Malicious employee accesses sensitive data.

**Mitigation**:
1. Role-based access control
2. Audit all admin actions
3. Separation of duties
4. Database access logging
5. KMS-based encryption (keys not accessible to admins)

---

## Security Testing

- **Penetration Testing**: Quarterly by certified testers
- **Vulnerability Scanning**: Weekly automated scans (Snyk, OWASP Dependency-Check)
- **Code Review**: Security review for all PRs
- **Threat Modeling**: Annual STRIDE analysis

---

See [Security Architecture](./security-architecture.md) for complete security controls.
