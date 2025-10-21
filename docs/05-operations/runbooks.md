# Runbooks

## Overview

Step-by-step operational procedures for common incidents and maintenance tasks.

---

## Incident Response

### 1. High Payment Failure Rate

**Symptoms**:
- Alert: `HighPaymentFailureRate`
- Dashboard shows spike in failed payments

**Diagnosis**:
```bash
# Check recent errors
kubectl logs -f deployment/wallet-service --tail=100 | grep ERROR

# Query failed transactions
SELECT status, COUNT(*) FROM transactions
WHERE created_at > SYSDATE - INTERVAL '1' HOUR
GROUP BY status;
```

**Resolution**:
1. Check payment gateway status
2. Review error patterns in logs
3. If gateway down: Enable circuit breaker bypass (manual approval mode)
4. Notify users of degraded service

---

### 2. Database Connection Pool Exhaustion

**Symptoms**:
- Alert: `DatabaseConnectionPoolExhausted`
- Slow query responses

**Diagnosis**:
```bash
# Check active connections
SELECT COUNT(*) FROM v$session WHERE username = 'WALLET_USER';

# Check HikariCP metrics
curl http://localhost:8081/actuator/metrics/hikaricp.connections.active
```

**Resolution**:
1. Identify long-running queries
2. Kill blocking sessions if necessary
3. Scale up application pods (increase connection pool size)
4. Review slow query logs

---

### 3. Cache Miss Spike

**Symptoms**:
- Alert: `CacheHitRateLow`
- Increased DB load

**Diagnosis**:
```bash
# Check Redis availability
redis-cli -h redis-cluster ping

# Check cache metrics
curl http://localhost:8081/actuator/metrics/cache.gets
```

**Resolution**:
1. Verify Redis cluster health
2. Check for cache key expiration policy changes
3. Pre-warm cache with common queries
4. Review cache invalidation logic

---

## Maintenance Tasks

### Rolling Restart

```bash
# Graceful rolling restart
kubectl rollout restart deployment/wallet-service -n wallet-production

# Monitor rollout
kubectl rollout status deployment/wallet-service -n wallet-production
```

### Database Migration

```bash
# Run Flyway migration
kubectl exec -it wallet-service-pod -- \
  java -jar flyway.jar migrate \
  -url=jdbc:oracle:thin:@db-host:1521/wallet \
  -user=wallet_user
```

### Scaling

```bash
# Manual scaling
kubectl scale deployment/wallet-service --replicas=10 -n wallet-production

# Update HPA limits
kubectl edit hpa wallet-service-hpa -n wallet-production
```

---

## Contacts

- **On-Call Engineer**: PagerDuty rotation
- **Database DBA**: db-team@company.com
- **Security Team**: security@company.com
