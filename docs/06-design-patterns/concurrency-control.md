# Concurrency Control

## Overview

Strategies for managing concurrent access to wallets and transactions ensuring data consistency and preventing race conditions.

---

## Optimistic Locking (Preferred)

**JPA Entity with Version**:
```java
@Entity
@Table(name = "wallets")
public class Wallet {

    @Id
    private UUID id;

    private BigDecimal balance;

    @Version  // Optimistic locking
    private Long version;

    // Other fields...
}
```

**Update Logic**:
```java
@Transactional
public void updateWalletBalance(UUID walletId, BigDecimal amount) {
    Wallet wallet = walletRepo.findById(walletId)
        .orElseThrow(() -> new WalletNotFoundException());

    wallet.setBalance(wallet.getBalance().add(amount));

    try {
        walletRepo.save(wallet);
    } catch (OptimisticLockingFailureException e) {
        // Retry or fail gracefully
        throw new ConcurrentUpdateException("Wallet was modified by another transaction");
    }
}
```

**Benefits**:
- No database locks held
- Better scalability
- Automatic conflict detection

---

## Pessimistic Locking (for Critical Operations)

**SELECT FOR UPDATE**:
```java
@Query("SELECT w FROM Wallet w WHERE w.id = :id FOR UPDATE")
Optional<Wallet> findByIdWithLock(@Param("id") UUID id);
```

**Usage**:
```java
@Transactional
public void processHighValuePayment(PaymentRequest request) {
    // Lock wallet during transaction
    Wallet wallet = walletRepo.findByIdWithLock(request.getWalletId())
        .orElseThrow();

    // Perform operations
    wallet.setBalance(wallet.getBalance().subtract(request.getAmount()));
    walletRepo.save(wallet);

    // Lock released at transaction end
}
```

---

## Distributed Locking (Redis)

For operations spanning multiple services:

```java
@Component
public class RedisDistributedLock {

    private final StringRedisTemplate redisTemplate;

    public boolean acquireLock(String resource, Duration timeout) {
        String lockKey = "lock:" + resource;
        Boolean acquired = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, "LOCKED", timeout);
        return Boolean.TRUE.equals(acquired);
    }

    public void releaseLock(String resource) {
        redisTemplate.delete("lock:" + resource);
    }

    @Around("@annotation(DistributedLock)")
    public Object executWithLock(ProceedingJoinPoint joinPoint) throws Throwable {
        String resource = getResourceKey(joinPoint);

        if (!acquireLock(resource, Duration.ofSeconds(30))) {
            throw new LockAcquisitionException("Could not acquire lock");
        }

        try {
            return joinPoint.proceed();
        } finally {
            releaseLock(resource);
        }
    }
}
```

---

## Retry Strategy

```java
@Retryable(
    value = {OptimisticLockingFailureException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 100, multiplier = 2)
)
public void updateWithRetry(UUID walletId, BigDecimal amount) {
    updateWalletBalance(walletId, amount);
}
```

---

See [Idempotency Pattern](./idempotency-deduplication.md) for duplicate prevention.
