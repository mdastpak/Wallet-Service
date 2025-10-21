# Chaos Engineering

## Overview

Chaos engineering practices to validate system resilience and identify weaknesses before they impact production.

---

## Chaos Experiments

### 1. Database Failure

**Hypothesis**: System should gracefully degrade when database is unavailable.

**Experiment**:
```bash
# Simulate database connection failure
kubectl exec -it postgres-0 -- pkill -9 postgres
```

**Expected Behavior**:
- Circuit breaker opens after 50% failure rate
- Health check transitions to unhealthy
- Kubernetes restarts unhealthy pod
- Recovery within 5 minutes

**Actual Results**:
- ✅ Circuit breaker activated after 23 failed requests
- ✅ Health check failed, pod restarted
- ✅ Service recovered in 3 minutes 12 seconds

---

### 2. Redis Cache Failure

**Hypothesis**: Application continues operating without cache, with degraded performance.

**Experiment**:
```bash
kubectl delete pod redis-0
```

**Expected Behavior**:
- Cache miss rate increases to 100%
- Queries fall back to database
- Latency increases but requests succeed
- New Redis pod starts and cache rebuilds

**Actual Results**:
- ✅ All requests succeeded (no errors)
- ⚠️ Latency increased from 10ms (p95) to 52ms (p95)
- ✅ Cache recovered after 90 seconds

---

### 3. Payment Gateway Timeout

**Hypothesis**: Circuit breaker prevents cascading failures when gateway is slow.

**Experiment**:
```java
@Component
public class ChaosPaymentGateway {
    public GatewayResponse process(PaymentRequest request) {
        Thread.sleep(10000);  // 10 second delay
        return response;
    }
}
```

**Expected Behavior**:
- Requests timeout after 5 seconds
- Circuit breaker opens after 50% failure rate
- Fallback to manual review mode
- Gateway recovers when latency normalizes

**Actual Results**:
- ✅ Circuit breaker opened after 52 timeouts
- ✅ Fallback activated, payments queued for manual review
- ✅ Circuit half-open after 60 seconds
- ✅ Full recovery after gateway fixed

---

### 4. Network Partition

**Hypothesis**: System handles network partitions between services gracefully.

**Experiment**:
```bash
# Block traffic to Kafka
iptables -A OUTPUT -d kafka-service -j DROP
```

**Expected Behavior**:
- Event publishing fails
- Outbox pattern queues events
- Events published when network recovers

**Actual Results**:
- ✅ Kafka producer retries for 30 seconds
- ✅ Events saved to outbox table
- ✅ Published successfully after partition healed
- ⚠️ Aggregation lag: 5 minutes during partition

---

### 5. Pod Crash (Random Kill)

**Hypothesis**: Kubernetes automatically recovers from pod crashes.

**Experiment**:
```bash
kubectl delete pod wallet-service-$(shuf -i 0-2 -n 1) -n wallet-production
```

**Expected Behavior**:
- In-flight requests to killed pod fail
- Kubernetes schedules new pod
- Service mesh routes to healthy pods
- Zero downtime for users

**Actual Results**:
- ✅ 0.02% request failure (in-flight requests)
- ✅ New pod ready in 45 seconds
- ✅ Service availability maintained

---

## Chaos Monkey Configuration

```yaml
chaos:
  monkey:
    enabled: true
    watcher:
      repository: true
      service: true
      restController: true
    assaults:
      level: 3  # Moderate chaos
      latency-active: true
      latency-range-start: 500
      latency-range-end: 3000
      exception-active: true
      exception-rate: 0.05  # 5% of requests
      kill-application-active: false  # Don't kill in production
      memory-active: false
```

---

## Game Days (Quarterly)

**Schedule**: Last Friday of quarter (9 AM - 12 PM)

**Participants**: Engineering, SRE, Product

**Scenarios**:
1. Database primary failure (failover test)
2. Region-wide outage (multi-region DR)
3. Kafka cluster failure (event replay)
4. Redis cluster failure (cache rebuild)
5. Payment gateway outage (manual mode)

**Success Criteria**:
- RTO < 5 minutes for all scenarios
- RPO = 0 (no data loss)
- All runbook procedures validated
- Lessons learned documented

---

## Monitoring During Chaos

```promql
# Error rate spike detection
rate(http_server_requests_seconds_count{status=~"5.."}[5m]) > 0.01

# Latency degradation
histogram_quantile(0.95, rate(wallet_payments_duration_seconds_bucket[5m])) > 0.2

# Circuit breaker state
resilience4j_circuitbreaker_state{name="paymentGateway"} == 1  # Open
```

---

## Lessons Learned

### Improvement 1: Cache Warming
**Issue**: Cache miss storm after Redis restart caused latency spike.
**Fix**: Implemented cache warming for top 1000 wallets on startup.

### Improvement 2: Connection Timeout
**Issue**: Database connection pool exhaustion during load.
**Fix**: Reduced max connection lifetime from 30min to 15min.

### Improvement 3: Outbox Cleanup
**Issue**: Outbox table grew indefinitely, slowing queries.
**Fix**: Added cleanup job to archive events older than 7 days.

---

See [Disaster Recovery](../05-operations/disaster-recovery.md) for recovery procedures.
