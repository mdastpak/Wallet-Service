# Audit Trail

## Overview

Comprehensive, immutable audit trail for all financial operations ensuring accountability and compliance.

---

## Audit Log Schema

```sql
CREATE TABLE audit_logs (
    id RAW(16) PRIMARY KEY,
    event_type VARCHAR2(50) NOT NULL,
    entity_type VARCHAR2(50) NOT NULL,
    entity_id RAW(16) NOT NULL,
    user_id RAW(16),
    action VARCHAR2(50) NOT NULL,
    old_value CLOB,
    new_value CLOB,
    correlation_id VARCHAR2(100),
    ip_address VARCHAR2(45),
    user_agent VARCHAR2(500),
    created_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL
);
```

**Characteristics**:
- **Immutable**: INSERT-only (no UPDATE or DELETE)
- **Comprehensive**: All state changes logged
- **Correlation**: Distributed tracing via correlation_id
- **PII Masked**: Sensitive data masked in logs

---

## Event Types

| Event Type | Description | Example |
|------------|-------------|---------|
| `TRANSACTION_CREATED` | New payment initiated | Payment of $100 created |
| `TRANSACTION_CONFIRMED` | Payment confirmed | Payment confirmed by gateway |
| `TRANSACTION_ROLLED_BACK` | Payment reversed | Admin rollback executed |
| `WALLET_CREATED` | New wallet created | USD wallet created |
| `WALLET_FROZEN` | Wallet frozen | Suspicious activity detected |
| `USER_LOGIN` | User authentication | User logged in from IP |
| `DISCOUNT_APPLIED` | Discount code used | Code SAVE20 applied |

---

## Audit Queries

### User Transaction History
```sql
SELECT
    al.created_at,
    al.event_type,
    al.action,
    al.new_value,
    al.ip_address
FROM audit_logs al
WHERE al.user_id = :user_id
ORDER BY al.created_at DESC;
```

### Compliance Report (All Transactions in Period)
```sql
SELECT
    t.id,
    t.created_at,
    t.amount,
    t.status,
    al.event_type,
    al.user_id,
    al.ip_address
FROM transactions t
JOIN audit_logs al ON t.id = al.entity_id
WHERE al.created_at BETWEEN :start_date AND :end_date
  AND al.entity_type = 'TRANSACTION';
```

---

## Retention & Archival

- **Active Logs**: 1 year in primary database
- **Archive**: 7 years in cold storage (S3 Glacier)
- **Compliance**: Meets SOX, PCI-DSS, GDPR requirements

---

See [Security Architecture](./security-architecture.md) for audit logging implementation.
