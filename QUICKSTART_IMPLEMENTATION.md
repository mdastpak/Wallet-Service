# Wallet Service - Quick Start Implementation Guide

**For**: Development Teams Starting Implementation
**Version**: 3.1
**Last Updated**: October 2025

---

## 🚀 Getting Started in 5 Minutes

### What Is This Project?

Production-ready **Wallet Service** design for enterprise financial platform supporting:
- Multi-currency wallets (fiat + crypto)
- Multi-business & hierarchical structures
- 4 payment types (Prepayment, Postpayment, Valuable, Credit)
- PCI-DSS & GDPR compliant

**Current Status**: 90% implementation-ready with comprehensive documentation

---

## 📋 Implementation Checklist

### Day 1: Setup & Understanding

- [ ] Read [Project Overview](./README.md)
- [ ] Review [System Architecture](./docs/01-architecture/system-overview.md)
- [ ] Understand [Database Schema](./docs/02-domain-model/database-schema.md)
- [ ] Check [Technology Stack](./docs/01-architecture/technology-stack.md)

### Week 1: Core Implementation

- [ ] Set up development environment
- [ ] Create database (Oracle Database 26ai) with schema
- [ ] Implement **Fraud Detection** ([spec](./docs/04-security-compliance/fraud-detection.md))
- [ ] Implement **Rate Limiting** ([spec](./docs/04-security-compliance/rate-limiting-specs.md))
- [ ] Add **Circuit Breaker** ([pattern](./docs/06-design-patterns/circuit-breaker.md))
- [ ] Add **Outbox Pattern** ([pattern](./docs/06-design-patterns/outbox-pattern.md))

### Week 2-3: Features & Integration

- [ ] Implement payment processing
- [ ] Add wallet management
- [ ] Integrate payment gateway
- [ ] Set up monitoring & alerts
- [ ] Load testing

### Week 4: Production Prep

- [ ] Security audit
- [ ] Disaster recovery testing
- [ ] Documentation review
- [ ] Staged rollout (10% → 50% → 100%)

---

## 🎯 Critical Documents (Must Read)

### 1. Implementation Roadmap
**File**: [IMPLEMENTATION_PLAN.md](./IMPLEMENTATION_PLAN.md)

**What**: Complete 3-week plan with priorities, resource allocation, timeline

**When to Read**: Before starting (Day 0)

**Key Sections**:
- Document structure reorganization
- Critical documentation gaps
- Week-by-week roadmap
- Success criteria

---

### 2. Improvements Summary
**File**: [IMPROVEMENTS_SUMMARY.md](./IMPROVEMENTS_SUMMARY.md)

**What**: Summary of all improvements made to reach 90% implementation readiness

**When to Read**: After implementation plan (Day 0)

**Key Sections**:
- What was done (Phase 1)
- What's pending (Phase 2)
- Success metrics
- Risk assessment

---

### 3. Fraud Detection System
**File**: [docs/04-security-compliance/fraud-detection.md](./docs/04-security-compliance/fraud-detection.md)

**What**: Complete fraud detection specification (rules + ML)

**When to Read**: Week 1 (before implementing payments)

**Implementation Priority**: 🔴 **P0 (CRITICAL)**

**What You'll Build**:
- Rule-based detection engine (25+ rules)
- XGBoost ML model (35 features)
- Real-time scoring system (<100ms)
- Manual review queue

**Effort**: 1-2 weeks

---

### 4. Rate Limiting Specifications
**File**: [docs/04-security-compliance/rate-limiting-specs.md](./docs/04-security-compliance/rate-limiting-specs.md)

**What**: Per-endpoint rate limits with sliding window algorithm

**When to Read**: Week 1 (before exposing APIs)

**Implementation Priority**: 🔴 **P0 (CRITICAL)**

**What You'll Build**:
- Redis-based rate limiter
- Per-endpoint, per-user, per-IP limits
- User tier system (5 tiers)
- Quota management

**Effort**: 2-3 days

---

### 5. Circuit Breaker Pattern
**File**: [docs/06-design-patterns/circuit-breaker.md](./docs/06-design-patterns/circuit-breaker.md)

**What**: Prevent cascading failures with graceful degradation

**When to Read**: Week 1 (before integrating external services)

**Implementation Priority**: 🔴 **P0 (CRITICAL)**

**What You'll Build**:
- Resilience4j circuit breakers
- Fallback mechanisms
- Health checks
- Monitoring dashboards

**Effort**: 2-3 days

---

### 6. Outbox Pattern
**File**: [docs/06-design-patterns/outbox-pattern.md](./docs/06-design-patterns/outbox-pattern.md)

**What**: Reliable event publishing with at-least-once guarantees

**When to Read**: Week 1 (before implementing Kafka integration)

**Implementation Priority**: 🔴 **P0 (CRITICAL)**

**What You'll Build**:
- Outbox events table
- Polling-based publisher
- Event ordering guarantees
- Failure handling

**Effort**: 3-4 days

---

## 📚 Complete Documentation Index

### Architecture
```
docs/01-architecture/
├── system-overview.md
├── technology-stack.md
├── data-flows.md
└── deployment-architecture.md
```

### Domain Model
```
docs/02-domain-model/
├── database-schema.md ⭐ Core database DDL
├── entity-relationships.md
├── ledger-model.md ⭐ Double-entry bookkeeping
├── payment-types.md ⭐ 4 payment types explained
└── multi-business-wallets.md
```

### API Specification
```
docs/03-api-specification/
├── payment-apis.md ⭐ Payment endpoints
├── wallet-apis.md ⭐ Wallet management
├── user-apis.md
└── openapi-spec.yaml
```

### Security & Compliance
```
docs/04-security-compliance/
├── fraud-detection.md 🆕 ⭐ Fraud system spec
├── rate-limiting-specs.md 🆕 ⭐ Rate limits
├── security-architecture.md
├── threat-model.md
└── audit-trail.md
```

### Operations
```
docs/05-operations/
├── deployment-guide.md
├── monitoring-alerting.md
└── runbooks.md
```

### Design Patterns
```
docs/06-design-patterns/
├── circuit-breaker.md 🆕 ⭐ Circuit breaker
├── outbox-pattern.md 🆕 ⭐ Outbox pattern
├── idempotency-deduplication.md ⭐ Idempotency
└── money-handling.md
```

### Integrations
```
docs/07-integrations/
├── kafka-integration.md
├── redis-integration.md
└── third-party-apis.md
```

### Testing
```
docs/08-testing/
├── testing-strategy.md
├── integration-tests.md
└── load-testing.md
```

**Legend**: ⭐ = Must read | 🆕 = Newly added

---

## 🏗️ Architecture Quick Reference

### Technology Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| **Backend** | Spring Boot + Java | 3.x / 17 LTS |
| **Database** | Oracle Database 26ai | 26ai |
| **Cache** | Redis Cluster | 7.x |
| **Messaging** | Apache Kafka | 3.x |
| **Container** | Kubernetes | 1.27+ |
| **Monitoring** | Prometheus + Grafana | Latest |

### Core Entities

```
BUSINESS (parent-child hierarchy)
  └─> USER (employees/consumers)
      └─> WALLET (per currency, per business)
          └─> TRANSACTION
              └─> LEDGER_ENTRY (double-entry)
```

### Payment Flow (Simplified)

```
1. User initiates payment → POST /v1/payments
2. Fraud check → Score < 0.3 = approve
3. Rate limit check → Within limits
4. Create transaction (PENDING)
5. Reserve amount in source wallet
6. Create ledger entries (RESERVED)
7. Circuit breaker → Call payment gateway
8. Confirm transaction (CONFIRMED)
9. Publish event to Kafka (via outbox)
10. Invalidate cache
11. Return success to user
```

---

## ⚡ Quick Implementation Examples

### 1. Create Fraud Rule

```java
@Component
public class FraudDetectionRules {

    @Rule(name = "HIGH_VELOCITY", threshold = "MEDIUM")
    public FraudScore checkVelocity(Transaction tx) {
        long recentCount = transactionRepository.countByUserIdAndCreatedAtAfter(
            tx.getUserId(),
            ZonedDateTime.now().minusMinutes(10)
        );

        if (recentCount > 5) {
            return FraudScore.builder()
                .flag("VELOCITY_ANOMALY")
                .score(0.3)
                .requireAdditionalVerification(true)
                .build();
        }

        return FraudScore.safe();
    }
}
```

### 2. Add Rate Limiting

```java
@RestController
public class PaymentController {

    @PostMapping("/v1/payments")
    @RateLimit(limit = 100, window = "1m", scope = "user")
    public PaymentResult createPayment(@RequestBody PaymentRequest request) {
        // Implementation
    }
}
```

### 3. Configure Circuit Breaker

```yaml
# application.yml
resilience4j.circuitbreaker:
  instances:
    paymentGateway:
      slidingWindowSize: 100
      failureRateThreshold: 50
      waitDurationInOpenState: 30s
      permittedNumberOfCallsInHalfOpenState: 3
```

### 4. Save with Outbox

```java
@Transactional
public Transaction createPayment(PaymentRequest request) {
    // 1. Save transaction
    Transaction tx = transactionRepository.save(...);

    // 2. Save outbox event (same transaction!)
    OutboxEvent event = OutboxEvent.builder()
        .aggregateType("TRANSACTION")
        .aggregateId(tx.getId())
        .eventType("TRANSACTION_CREATED")
        .eventData(toJson(tx))
        .build();

    outboxEventRepository.save(event);

    return tx;  // Both saved atomically!
}
```

---

## 🔍 Troubleshooting Guide

### Common Issues

#### 1. "Circuit breaker is OPEN, requests rejected"

**Cause**: Payment gateway is down or slow

**Fix**:
```bash
# Check circuit breaker state
curl localhost:8080/actuator/circuitbreakers

# If stuck open, check gateway health
curl https://gateway.example.com/health

# Reset circuit breaker (emergency only)
curl -X POST localhost:8080/admin/circuit-breaker/paymentGateway/reset
```

#### 2. "Outbox backlog growing"

**Cause**: Kafka down or publisher not running

**Fix**:
```sql
-- Check backlog size
SELECT COUNT(*) FROM outbox_events WHERE published = 0;

-- Check oldest unpublished
SELECT MIN(created_at) FROM outbox_events WHERE published = 0;

-- If Kafka is down, events will accumulate
-- They'll publish automatically when Kafka recovers
```

#### 3. "Rate limit exceeded"

**Cause**: User/IP hitting limit

**Fix**:
```bash
# Check user's current rate limit status
curl localhost:8080/v1/users/{id}/rate-limit-status

# Response:
{
  "limit": 100,
  "remaining": 0,
  "reset_at": "2025-10-21T10:35:00Z",
  "retry_after_seconds": 42
}

# Temporarily increase limit (admin only)
curl -X POST localhost:8080/admin/rate-limits/user/{id}/increase \
  -d '{"newLimit": 200, "duration": "1h"}'
```

---

## 📊 Success Metrics

### Implementation Progress

Track your progress:

- [ ] Database schema created
- [ ] Core entities implemented
- [ ] Fraud detection working
- [ ] Rate limiting active
- [ ] Circuit breakers configured
- [ ] Outbox pattern publishing events
- [ ] APIs responding
- [ ] Monitoring dashboards live
- [ ] Load testing passed
- [ ] Security audit completed

### Performance Targets

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Payment latency (p95) | <100ms | Prometheus: `histogram_quantile(0.95, payment_duration_seconds)` |
| Balance query (p95) | <10ms | Prometheus: `histogram_quantile(0.95, balance_query_duration_seconds)` |
| Throughput | >15,000 TPS | Load test with k6/JMeter |
| Cache hit rate | >95% | Redis INFO stats |
| Fraud detection rate | >70% | Manual validation of flagged transactions |
| False positive rate | <5% | User reports / total flagged |

---

## 🆘 Getting Help

### Documentation Issues

If documentation is unclear:
1. Check related documents (linked at bottom of each file)
2. Review examples in the document
3. Check this quick start guide

### Implementation Questions

Ask on these specific topics:
- **Fraud Detection**: See fraud-detection.md FAQ section
- **Rate Limiting**: See rate-limiting-specs.md troubleshooting
- **Circuit Breakers**: See circuit-breaker.md best practices
- **Outbox Pattern**: See outbox-pattern.md implementation guide

### Escalation Path

```
Level 1: Check documentation
  ↓ (if not found)
Level 2: Review implementation plan
  ↓ (if still unclear)
Level 3: Ask team lead
  ↓ (if architectural decision needed)
Level 4: Escalate to architect
```

---

## 🎓 Learning Path

### For New Team Members

**Week 1: Understand Domain**
- Read: README.md, system-overview.md, database-schema.md
- Understand: Wallets, transactions, ledger model
- Do: Set up local environment

**Week 2: Understand Patterns**
- Read: fraud-detection.md, circuit-breaker.md, outbox-pattern.md
- Understand: Why these patterns, how they prevent failures
- Do: Prototype one pattern

**Week 3: Implement Feature**
- Pick: One payment type or wallet operation
- Implement: With fraud check, rate limiting, circuit breaker
- Review: With senior engineer

**Week 4: Testing & Monitoring**
- Write: Integration tests
- Add: Monitoring metrics
- Run: Load test
- Review: Performance results

---

## ✅ Pre-Launch Checklist

### Security

- [ ] Fraud detection rules configured
- [ ] Rate limits set per endpoint
- [ ] Circuit breakers configured for all external services
- [ ] PCI-DSS compliance verified
- [ ] Penetration test completed

### Reliability

- [ ] Outbox pattern publishing events
- [ ] Idempotency working (duplicate protection)
- [ ] Circuit breakers tested (manual failover)
- [ ] Disaster recovery tested
- [ ] Backup/restore verified

### Performance

- [ ] Load test: 15K TPS sustained for 1 hour
- [ ] Cache hit rate: >95%
- [ ] p95 latency: <100ms for payments
- [ ] Database indexes optimized
- [ ] Connection pools tuned

### Monitoring

- [ ] Prometheus scraping metrics
- [ ] Grafana dashboards configured
- [ ] Alerts set up (PagerDuty/Opsgenie)
- [ ] Log aggregation working (ELK)
- [ ] Distributed tracing enabled

### Compliance

- [ ] Audit trail logging everything
- [ ] PII masked in logs
- [ ] GDPR data export working
- [ ] Regulatory reporting configured
- [ ] Data retention policies enforced

---

## 📝 Final Notes

### What Makes This Documentation Special

1. **Implementation-Ready**: Not just theory, actual specs you can code from
2. **Comprehensive**: 90% coverage of production scenarios
3. **Battle-Tested Patterns**: Circuit breaker, outbox, fraud detection
4. **Examples Included**: Every document has code examples
5. **Monitoring Built-In**: Metrics and alerts specified

### What's Still Pending (Phase 2)

- Payment gateway integration guide (enhanced)
- Disaster recovery runbook
- KYC/AML workflow
- Capacity planning guide
- Monitoring dashboards (detailed)
- Performance optimization guide

**Timeline**: 2 more weeks to reach 95% readiness

### Final Recommendation

**Start Here**:
1. Read this guide (you are here! ✓)
2. Read implementation plan
3. Read improvements summary
4. Pick one P0 item and implement
5. Review with team

**Don't Skip**:
- Fraud detection (will lose money without it)
- Rate limiting (will get DDoS-ed without it)
- Circuit breakers (will have cascading failures without it)
- Outbox pattern (will lose events without it)

**Good Luck! 🚀**

---

**Prepared By**: Claude (AI Assistant)
**For**: Wallet Service Implementation Team
**Status**: Ready to Start Implementation

**Questions?** Refer to specific document or escalate per above guide.
