# Logging Strategy

## Overview

Structured logging with PII masking, centralized aggregation (ELK), and retention policies.

---

## Log Levels

| Level | Usage | Examples |
|-------|-------|----------|
| **ERROR** | System failures, exceptions | Payment gateway timeout, DB connection lost |
| **WARN** | Degraded performance, recoverable | Cache miss, retry attempt |
| **INFO** | Business events | Payment created, wallet created |
| **DEBUG** | Detailed flow (non-production) | SQL queries, Redis commands |
| **TRACE** | Very detailed (development only) | Method entry/exit |

---

## PII Masking

```java
@Component
public class PiiMasker {

    public String maskEmail(String email) {
        String[] parts = email.split("@");
        return parts[0].substring(0, 2) + "***@" + parts[1];
    }

    public String maskPhone(String phone) {
        return phone.replaceAll("\\d(?=\\d{4})", "*");
    }
}
```

**Example**:
- Email: `user@example.com` → `us***@example.com`
- Phone: `+1234567890` → `+1*****7890`

---

## Structured Logging (JSON)

```json
{
  "timestamp": "2025-01-15T10:30:00Z",
  "level": "INFO",
  "logger": "PaymentService",
  "message": "Payment processed successfully",
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "transactionId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "amount": 10000,
  "currency": "USD",
  "correlationId": "abc-123-xyz"
}
```

---

## Log Aggregation

**ELK Stack**:
- **Elasticsearch**: Storage and search
- **Logstash**: Processing and enrichment
- **Kibana**: Visualization and dashboards

**Retention**: 30 days active, 365 days archived

---

See [Monitoring & Alerting](./monitoring-alerting.md) for complete logging implementation.
