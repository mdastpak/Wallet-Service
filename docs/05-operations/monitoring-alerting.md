# Monitoring & Alerting

## Overview

Comprehensive observability stack using Prometheus, Grafana, and OpenTelemetry for real-time monitoring and alerting.

---

## Metrics Collection (Prometheus)

### Application Metrics

**Spring Boot Actuator + Micrometer**:
```java
@Component
public class WalletMetrics {

    private final Counter paymentCounter;
    private final Timer paymentTimer;
    private final Gauge activeWallets;

    public WalletMetrics(MeterRegistry registry) {
        // Payment counters
        this.paymentCounter = Counter.builder("wallet.payments.total")
            .description("Total number of payments processed")
            .tags("type", "payment")
            .register(registry);

        // Payment latency
        this.paymentTimer = Timer.builder("wallet.payments.duration")
            .description("Payment processing duration")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);

        // Active wallets gauge
        this.activeWallets = Gauge.builder("wallet.active.count")
            .description("Number of active wallets")
            .register(registry, this, m -> walletRepo.countByStatus("ACTIVE"));
    }
}
```

---

### Key Metrics

| Metric | Type | Description | Alert Threshold |
|--------|------|-------------|-----------------|
| `wallet_payments_total` | Counter | Total payments processed | - |
| `wallet_payments_duration_seconds` | Histogram | Payment processing time | p95 > 200ms |
| `wallet_payments_failed_total` | Counter | Failed payments | rate > 1% |
| `wallet_balance_queries_total` | Counter | Balance queries | - |
| `wallet_cache_hit_ratio` | Gauge | Redis cache hit rate | < 80% |
| `wallet_db_connections_active` | Gauge | Active DB connections | > 80% pool |
| `wallet_idempotency_conflicts_total` | Counter | Duplicate requests | spike > 2x baseline |

---

## Dashboards (Grafana)

### Main Dashboard Panels

1. **Request Rate** (req/sec)
```promql
rate(http_server_requests_seconds_count{job="wallet-service"}[5m])
```

2. **Payment Processing Latency** (p95, p99)
```promql
histogram_quantile(0.95,
  rate(wallet_payments_duration_seconds_bucket[5m])
)
```

3. **Error Rate**
```promql
rate(http_server_requests_seconds_count{status=~"5.."}[5m])
/ rate(http_server_requests_seconds_count[5m]) * 100
```

4. **Cache Hit Ratio**
```promql
rate(wallet_cache_hits_total[5m])
/ (rate(wallet_cache_hits_total[5m]) + rate(wallet_cache_misses_total[5m])) * 100
```

5. **Database Connection Pool**
```promql
hikaricp_connections_active{pool="walletServicePool"}
```

---

## Alerting Rules

### Critical Alerts (PagerDuty)

```yaml
groups:
  - name: wallet_critical
    interval: 30s
    rules:
      - alert: HighPaymentFailureRate
        expr: |
          rate(wallet_payments_failed_total[5m])
          / rate(wallet_payments_total[5m]) > 0.01
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Payment failure rate > 1%"
          description: "{{ $value | humanizePercentage }} of payments are failing"

      - alert: PaymentProcessingSlowness
        expr: |
          histogram_quantile(0.95,
            rate(wallet_payments_duration_seconds_bucket[5m])
          ) > 0.2
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Payment processing p95 latency > 200ms"

      - alert: ServiceDown
        expr: up{job="wallet-service"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Wallet Service is down"

      - alert: DatabaseConnectionPoolExhausted
        expr: |
          hikaricp_connections_active / hikaricp_connections_max > 0.9
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "DB connection pool > 90% utilized"
```

---

### Warning Alerts (Slack)

```yaml
      - alert: CacheHitRateLow
        expr: |
          wallet_cache_hit_ratio < 0.8
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Cache hit ratio < 80%"

      - alert: HighIdempotencyConflicts
        expr: |
          rate(wallet_idempotency_conflicts_total[5m]) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High rate of idempotency conflicts (>10/sec)"
```

---

## Distributed Tracing (OpenTelemetry)

### Configuration

```yaml
management:
  tracing:
    sampling:
      probability: 0.1  # Sample 10% of requests
  otlp:
    tracing:
      endpoint: http://jaeger:4318/v1/traces
```

### Trace Context Propagation

```java
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                   HttpServletResponse response,
                                   FilterChain chain) {
        String correlationId = request.getHeader("X-Correlation-ID");
        if (correlationId == null) {
            correlationId = UUID.randomUUID().toString();
        }

        MDC.put("correlationId", correlationId);
        response.setHeader("X-Correlation-ID", correlationId);

        // Add to current span
        Span.current().setAttribute("correlation.id", correlationId);

        try {
            chain.doFilter(request, response);
        } finally {
            MDC.remove("correlationId");
        }
    }
}
```

---

## Logging (ELK Stack)

### Structured Logging

```java
@Slf4j
@Service
public class PaymentService {

    public PaymentResponse processPayment(PaymentRequest request) {
        log.info("Processing payment",
            kv("userId", request.getUserId()),
            kv("amount", request.getAmount()),
            kv("currency", request.getCurrency()),
            kv("paymentType", request.getPaymentType()),
            kv("correlationId", MDC.get("correlationId"))
        );

        try {
            // Process payment
            return response;
        } catch (Exception e) {
            log.error("Payment processing failed",
                kv("userId", request.getUserId()),
                kv("error", e.getMessage()),
                e
            );
            throw e;
        }
    }
}
```

**Logback Configuration** (JSON format):
```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>correlationId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>
        </encoder>
    </appender>
</configuration>
```

---

## Health Checks

### Liveness Probe
```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8081
  initialDelaySeconds: 60
  periodSeconds: 10
```

### Readiness Probe
```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8081
  initialDelaySeconds: 30
  periodSeconds: 5
```

### Custom Health Indicators

```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        try {
            long count = transactionRepo.count();
            return Health.up()
                .withDetail("transactions", count)
                .build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```

---

## On-Call Runbooks

See [Runbooks](./runbooks.md) for detailed incident response procedures.

---

## Performance SLOs

| Metric | Target | Measurement |
|--------|--------|-------------|
| **Availability** | 99.99% | Uptime monitoring |
| **Payment Latency (p95)** | < 100ms | Histogram |
| **Cache Hit Rate** | > 85% | Gauge |
| **Error Rate** | < 0.1% | Counter ratio |

---

See [Deployment Architecture](../01-architecture/deployment-architecture.md) for Kubernetes monitoring setup.
