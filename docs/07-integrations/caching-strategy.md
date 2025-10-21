# Caching Strategy (Redis)

## Overview

Redis-based caching for balance queries, idempotency enforcement, and distributed locking.

---

## Cache Use Cases

| Use Case | Data Structure | TTL | Invalidation |
|----------|---------------|-----|--------------|
| **Wallet Balance** | String | 5 min | Event-driven |
| **Idempotency Lock** | String (NX PX) | Configurable | Explicit delete |
| **Response Cache** | String | 1 hour | TTL expiry |
| **Rate Limiting** | String (INCR) | 1 minute | TTL expiry |
| **Session Storage** | Hash | 30 min | Logout/TTL |
| **User Aggregates** | Hash | 5 min | Event-driven |

---

## Balance Caching

```java
@Service
public class WalletBalanceCache {

    private final StringRedisTemplate redisTemplate;
    private static final Duration TTL = Duration.ofMinutes(5);

    public Optional<BigDecimal> getCachedBalance(UUID walletId) {
        String key = buildKey(walletId);
        String cached = redisTemplate.opsForValue().get(key);

        if (cached != null) {
            return Optional.of(new BigDecimal(cached));
        }
        return Optional.empty();
    }

    public void cacheBalance(UUID walletId, BigDecimal balance) {
        String key = buildKey(walletId);
        redisTemplate.opsForValue().set(key, balance.toString(), TTL);
    }

    public void invalidate(UUID walletId) {
        String key = buildKey(walletId);
        redisTemplate.delete(key);
    }

    private String buildKey(UUID walletId) {
        return "wallet:balance:" + walletId;
    }
}
```

---

## Cache-Aside Pattern

```java
@Service
public class BalanceService {

    public BigDecimal getBalance(UUID walletId) {
        // Try cache first
        return balanceCache.getCachedBalance(walletId)
            .orElseGet(() -> {
                // Cache miss - compute from DB
                BigDecimal balance = computeBalanceFromLedger(walletId);

                // Update cache
                balanceCache.cacheBalance(walletId, balance);

                return balance;
            });
    }

    private BigDecimal computeBalanceFromLedger(UUID walletId) {
        return ledgerRepo.sumBalance(walletId);
    }
}
```

---

## Event-Driven Invalidation

```java
@Service
public class CacheInvalidationListener {

    @KafkaListener(topics = "wallet.transactions.v1")
    public void handleTransactionEvent(TransactionEvent event) {
        // Invalidate affected wallet caches
        balanceCache.invalidate(event.getSourceWalletId());
        balanceCache.invalidate(event.getDestinationWalletId());

        // Invalidate user aggregate
        String userKey = "user:balance:" + event.getUserId();
        redisTemplate.delete(userKey);
    }
}
```

---

## Redis Cluster Configuration

```yaml
spring:
  redis:
    cluster:
      nodes:
        - redis-0.redis:6379
        - redis-1.redis:6379
        - redis-2.redis:6379
      max-redirects: 3
    timeout: 2000ms
    lettuce:
      pool:
        max-active: 50
        max-idle: 20
        min-idle: 5
        max-wait: 1000ms
```

---

## Cache Monitoring

```java
@Component
public class CacheMetrics {

    private final Counter cacheHits;
    private final Counter cacheMisses;

    public CacheMetrics(MeterRegistry registry) {
        this.cacheHits = Counter.builder("wallet.cache.hits")
            .description("Cache hit count")
            .register(registry);

        this.cacheMisses = Counter.builder("wallet.cache.misses")
            .description("Cache miss count")
            .register(registry);
    }

    public void recordHit() {
        cacheHits.increment();
    }

    public void recordMiss() {
        cacheMisses.increment();
    }
}
```

---

## Cache Prewarming

```java
@Service
public class CachePrewarmingService {

    @Scheduled(cron = "0 0 6 * * *")  // Daily at 6 AM
    public void prewarmTopWallets() {
        List<UUID> topWallets = walletRepo.findTopActiveWallets(1000);

        for (UUID walletId : topWallets) {
            BigDecimal balance = computeBalanceFromLedger(walletId);
            balanceCache.cacheBalance(walletId, balance);
        }

        log.info("Prewarmed {} wallet balances", topWallets.size());
    }
}
```

---

See [Idempotency Pattern](../06-design-patterns/idempotency-deduplication.md) for Redis lock implementation.
