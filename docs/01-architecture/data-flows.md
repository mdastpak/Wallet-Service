# Data Flows

## Overview

This document details the complete data flows for all critical operations in the Wallet Service, including payment processing, rollback handling, balance aggregation, and idempotency management.

---

## 1. Payment Processing Flow

### 1.1 Standard Payment (Happy Path)

```mermaid
sequenceDiagram
    participant Client
    participant API Gateway
    participant Payment Service
    participant Redis
    participant Oracle DB
    participant Kafka
    participant Payment Gateway
    participant Aggregation Service

    Client->>API Gateway: POST /v1/payments<br/>Idempotency-Key: xxx<br/>Authorization: Bearer token

    API Gateway->>API Gateway: Validate JWT Token
    API Gateway->>API Gateway: Extract user_id, business_id

    API Gateway->>Payment Service: Forward Request + Claims

    Note over Payment Service,Redis: Idempotency Check
    Payment Service->>Redis: SET NX PX idempotency:xxx TTL=5m

    alt Lock Acquired (First Request)
        Payment Service->>Oracle DB: BEGIN TRANSACTION

        Note over Payment Service,Oracle DB: Create Idempotency Record
        Payment Service->>Oracle DB: INSERT idempotency_keys<br/>(key, status=PENDING, user_id, endpoint)

        Note over Payment Service,Oracle DB: Validate Wallet & Balance
        Payment Service->>Oracle DB: SELECT wallet WHERE id=:wallet_id FOR UPDATE
        Payment Service->>Oracle DB: SELECT SUM(amount) FROM ledger_entries<br/>WHERE wallet_id=:wallet_id
        Payment Service->>Payment Service: Check balance >= payment amount

        Note over Payment Service,Oracle DB: Create Transaction Record
        Payment Service->>Oracle DB: INSERT transactions<br/>(id, user_id, business_id,<br/>amount, currency, type, status=PENDING)

        Note over Payment Service,Oracle DB: Create Ledger Entries (Double-Entry)
        Payment Service->>Oracle DB: INSERT ledger_entries<br/>(transaction_id, wallet_id, debit, amount)
        Payment Service->>Oracle DB: INSERT ledger_entries<br/>(transaction_id, destination, credit, amount)

        Payment Service->>Oracle DB: COMMIT

        Note over Payment Service,Payment Gateway: Process External Payment
        Payment Service->>Payment Gateway: POST /process<br/>amount, currency, metadata
        Payment Gateway-->>Payment Service: 200 OK {gateway_tx_id}

        Note over Payment Service,Oracle DB: Update to CONFIRMED
        Payment Service->>Oracle DB: UPDATE transactions<br/>SET status=CONFIRMED, gateway_tx_id=xxx
        Payment Service->>Oracle DB: UPDATE idempotency_keys<br/>SET status=CONFIRMED

        Note over Payment Service,Kafka: Emit Event
        Payment Service->>Kafka: Publish TransactionCompleted Event<br/>{tx_id, user_id, amount, timestamp}

        Note over Payment Service,Redis: Cache Response
        Payment Service->>Redis: SET response:xxx = {result} EX 3600

        Payment Service-->>Client: 200 OK<br/>{transaction_id, status, amount}

        Note over Kafka,Aggregation Service: Async Aggregation Update
        Kafka->>Aggregation Service: Consume TransactionCompleted
        Aggregation Service->>Redis: INCRBYFLOAT user:balance:123 {amount}
        Aggregation Service->>Redis: INCRBYFLOAT business:balance:456 {amount}

    else Lock Failed (Duplicate Request)
        Payment Service->>Oracle DB: SELECT FROM idempotency_keys<br/>WHERE key=xxx AND user_id=:user_id

        alt Status = CONFIRMED
            Payment Service->>Redis: GET response:xxx
            Payment Service-->>Client: 200 OK (Cached Response)
        else Status = PENDING
            Payment Service-->>Client: 409 Conflict<br/>{message: "Payment in progress"}
        else Not Found (expired)
            Payment Service-->>Client: 410 Gone<br/>{message: "Idempotency key expired"}
        end
    end
```

### 1.2 Payment Type-Specific Flows

#### PREPAYMENT Flow
```mermaid
sequenceDiagram
    participant Client
    participant Payment Service
    participant Oracle DB
    participant Wallet

    Client->>Payment Service: POST /payments<br/>type=PREPAYMENT
    Payment Service->>Oracle DB: BEGIN TRANSACTION

    Note over Payment Service: Reserve funds immediately
    Payment Service->>Oracle DB: INSERT ledger_entries<br/>(wallet_id, DEBIT, amount, status=RESERVED)
    Payment Service->>Oracle DB: INSERT transactions<br/>(type=PREPAYMENT, status=PENDING)
    Payment Service->>Oracle DB: COMMIT

    Note over Payment Service: Wait for order fulfillment
    Client->>Payment Service: POST /payments/{id}/capture

    Payment Service->>Oracle DB: UPDATE transactions<br/>SET status=CONFIRMED
    Payment Service->>Oracle DB: UPDATE ledger_entries<br/>SET status=CONFIRMED
    Payment Service-->>Client: 200 OK {captured}
```

#### POSTPAYMENT Flow
```mermaid
sequenceDiagram
    participant Client
    participant Payment Service
    participant Oracle DB
    participant Credit Service

    Client->>Payment Service: POST /payments<br/>type=POSTPAYMENT

    Note over Payment Service: Create invoice, no immediate debit
    Payment Service->>Oracle DB: INSERT transactions<br/>(type=POSTPAYMENT, status=INVOICED)
    Payment Service->>Oracle DB: INSERT credit_accounts<br/>(user_id, amount_due, due_date)
    Payment Service-->>Client: 200 OK {invoice_id}

    Note over Payment Service: Later: Capture payment
    Client->>Payment Service: POST /payments/{id}/settle
    Payment Service->>Oracle DB: INSERT ledger_entries (DEBIT)
    Payment Service->>Oracle DB: UPDATE credit_accounts<br/>SET status=PAID
```

#### VALUABLE (High-Value) Flow
```mermaid
sequenceDiagram
    participant Client
    participant Payment Service
    participant 2FA Service
    participant Risk Engine
    participant Oracle DB

    Client->>Payment Service: POST /payments<br/>type=VALUABLE<br/>amount > $10,000

    Payment Service->>Risk Engine: Evaluate Risk Score<br/>(user_id, amount, history)
    Risk Engine-->>Payment Service: {score: 85, approved: true}

    alt Risk Score OK
        Payment Service->>2FA Service: Request 2FA Code<br/>(user_id, phone)
        2FA Service-->>Client: SMS/Email with code

        Client->>Payment Service: POST /payments/{id}/confirm<br/>code=123456
        Payment Service->>2FA Service: Verify Code
        2FA Service-->>Payment Service: {valid: true}

        Payment Service->>Oracle DB: Process normal payment flow
        Payment Service-->>Client: 200 OK
    else Risk Score High
        Payment Service-->>Client: 403 Forbidden<br/>{reason: "Manual review required"}
    end
```

#### CREDIT (Installment) Flow
```mermaid
sequenceDiagram
    participant Client
    participant Payment Service
    participant Oracle DB
    participant Scheduler

    Client->>Payment Service: POST /payments<br/>type=CREDIT<br/>installments=12

    Payment Service->>Oracle DB: INSERT transactions<br/>(type=CREDIT, total_amount)
    Payment Service->>Oracle DB: INSERT credit_schedules<br/>(tx_id, installments=12)

    loop Each Installment
        Payment Service->>Oracle DB: INSERT installment_details<br/>(schedule_id, due_date, amount)
    end

    Payment Service-->>Client: 200 OK {schedule}

    Note over Scheduler: Monthly job
    Scheduler->>Oracle DB: SELECT overdue installments
    Scheduler->>Payment Service: Process installment payment
    Payment Service->>Oracle DB: INSERT ledger_entries (DEBIT)
```

---

## 2. Payment Rollback Flow

```mermaid
sequenceDiagram
    participant Admin
    participant API Gateway
    participant Rollback Service
    participant Oracle DB
    participant Kafka
    participant Discount Service
    participant Aggregation Service

    Admin->>API Gateway: POST /v1/payments/{id}/rollback<br/>Authorization: Bearer (ADMIN)
    API Gateway->>API Gateway: Validate Admin Role
    API Gateway->>Rollback Service: Forward Request

    Rollback Service->>Oracle DB: BEGIN TRANSACTION

    Note over Rollback Service,Oracle DB: Validate Original Transaction
    Rollback Service->>Oracle DB: SELECT FROM transactions<br/>WHERE id=:id FOR UPDATE
    Rollback Service->>Rollback Service: Check status=CONFIRMED<br/>Check not already rolled back

    Note over Rollback Service,Oracle DB: Create Compensation Transaction
    Rollback Service->>Oracle DB: INSERT transactions<br/>(type=ROLLBACK,<br/>ref_transaction_id=:id,<br/>amount=-original_amount,<br/>status=PENDING)

    Note over Rollback Service,Oracle DB: Create Reverse Ledger Entries
    Rollback Service->>Oracle DB: INSERT ledger_entries<br/>(transaction_id=rollback_id,<br/>wallet_id, CREDIT, amount)
    Rollback Service->>Oracle DB: INSERT ledger_entries<br/>(transaction_id=rollback_id,<br/>destination, DEBIT, amount)

    Note over Rollback Service,Oracle DB: Update Original Transaction
    Rollback Service->>Oracle DB: UPDATE transactions<br/>SET status=ROLLED_BACK<br/>WHERE id=:id

    Note over Rollback Service,Oracle DB: Update Rollback Transaction
    Rollback Service->>Oracle DB: UPDATE transactions<br/>SET status=CONFIRMED<br/>WHERE id=rollback_id

    Rollback Service->>Oracle DB: COMMIT

    Note over Rollback Service,Kafka: Emit Rollback Event
    Rollback Service->>Kafka: Publish TransactionRolledBack Event<br/>{tx_id, rollback_id, amount, reason}

    Rollback Service-->>Admin: 200 OK<br/>{rollback_transaction_id}

    Note over Kafka,Aggregation Service: Update Cached Balances
    Kafka->>Aggregation Service: Consume TransactionRolledBack
    Aggregation Service->>Redis: INCRBYFLOAT user:balance:123 {-amount}
    Aggregation Service->>Redis: INCRBYFLOAT business:balance:456 {-amount}
```

---

## 3. Balance Aggregation Flow

### 3.1 Cached Balance Query (Fast Path)

```mermaid
sequenceDiagram
    participant Client
    participant API Gateway
    participant Aggregation Service
    participant Redis

    Client->>API Gateway: GET /v1/aggregates/users/{id}/balance
    API Gateway->>Aggregation Service: Forward Request

    Aggregation Service->>Redis: GET user:balance:{id}

    alt Cache Hit
        Redis-->>Aggregation Service: {balance: 1500.00, currency: USD}
        Aggregation Service-->>Client: 200 OK<br/>{balance: 1500.00, cached: true}
    else Cache Miss
        Aggregation Service->>Aggregation Service: Trigger compute from DB
        Note over Aggregation Service: See 3.2 Computed Balance
    end
```

### 3.2 Computed Balance Query (Slow Path)

```mermaid
sequenceDiagram
    participant Aggregation Service
    participant Oracle DB
    participant Redis
    participant Client

    Note over Aggregation Service: Cache miss or refresh requested

    Aggregation Service->>Oracle DB: SELECT wallet_id, currency<br/>FROM wallets<br/>WHERE user_id=:id

    loop For Each Wallet
        Aggregation Service->>Oracle DB: SELECT SUM(CASE WHEN type='DEBIT' THEN -amount ELSE amount END)<br/>FROM ledger_entries<br/>WHERE wallet_id=:wallet_id<br/>AND status='CONFIRMED'
    end

    Aggregation Service->>Aggregation Service: Sum all wallet balances<br/>by currency

    Note over Aggregation Service,Redis: Cache Result
    Aggregation Service->>Redis: SET user:balance:{id} = {result}<br/>EX 300 (5 min TTL)

    Aggregation Service-->>Client: 200 OK<br/>{balance: 1500.00, cached: false}
```

### 3.3 Event-Driven Aggregation Update

```mermaid
sequenceDiagram
    participant Payment Service
    participant Kafka
    participant Aggregation Service
    participant Redis
    participant Oracle DB

    Payment Service->>Kafka: Publish TransactionCompleted<br/>{tx_id, user_id, business_id, amount}

    Kafka->>Aggregation Service: Consume Event

    Note over Aggregation Service: Incremental Update
    Aggregation Service->>Redis: INCRBYFLOAT user:balance:{user_id} {amount}
    Aggregation Service->>Redis: INCRBYFLOAT business:balance:{business_id} {amount}
    Aggregation Service->>Redis: INCRBYFLOAT global:balance {amount}

    Note over Aggregation Service,Oracle DB: Update Materialized View (Async)
    Aggregation Service->>Oracle DB: REFRESH MATERIALIZED VIEW user_balance_summary
```

---

## 4. Wallet Creation Flow

```mermaid
sequenceDiagram
    participant Client
    participant API Gateway
    participant Wallet Service
    participant Oracle DB
    participant Kafka

    Client->>API Gateway: POST /v1/wallets<br/>{user_id, currency: "USD"}
    API Gateway->>Wallet Service: Forward Request

    Wallet Service->>Oracle DB: BEGIN TRANSACTION

    Note over Wallet Service,Oracle DB: Validate User
    Wallet Service->>Oracle DB: SELECT FROM users<br/>WHERE id=:user_id FOR UPDATE

    Note over Wallet Service,Oracle DB: Check Duplicate
    Wallet Service->>Oracle DB: SELECT FROM wallets<br/>WHERE user_id=:user_id AND currency='USD'

    alt Wallet Exists
        Wallet Service->>Oracle DB: ROLLBACK
        Wallet Service-->>Client: 409 Conflict<br/>{error: "Wallet already exists"}
    else New Wallet
        Note over Wallet Service,Oracle DB: Create Wallet
        Wallet Service->>Oracle DB: INSERT wallets<br/>(id=uuid(), user_id, business_id,<br/>currency, created_at, status=ACTIVE)

        Wallet Service->>Oracle DB: COMMIT

        Note over Wallet Service,Kafka: Emit Event
        Wallet Service->>Kafka: Publish WalletCreated Event<br/>{wallet_id, user_id, currency}

        Wallet Service-->>Client: 201 Created<br/>{wallet_id, user_id, currency, balance: 0}
    end
```

---

## 5. Multi-Currency Balance Query Flow

```mermaid
sequenceDiagram
    participant Client
    participant Aggregation Service
    participant Oracle DB
    participant Redis
    participant Exchange Rate Service

    Client->>Aggregation Service: GET /v1/users/{id}/balance?currency=USD

    Note over Aggregation Service,Oracle DB: Get All Wallets
    Aggregation Service->>Oracle DB: SELECT wallet_id, currency<br/>FROM wallets WHERE user_id=:id

    loop For Each Wallet
        Aggregation Service->>Redis: GET wallet:balance:{wallet_id}

        alt Cache Hit
            Redis-->>Aggregation Service: {balance: X, currency: EUR}
        else Cache Miss
            Aggregation Service->>Oracle DB: SUM ledger_entries
            Aggregation Service->>Redis: SET wallet:balance (cache)
        end
    end

    Note over Aggregation Service: Convert to Target Currency

    loop For Each Non-USD Balance
        Aggregation Service->>Exchange Rate Service: GET /rates?from=EUR&to=USD
        Exchange Rate Service-->>Aggregation Service: {rate: 1.08}
        Aggregation Service->>Aggregation Service: Convert EUR to USD
    end

    Aggregation Service->>Aggregation Service: Sum all in USD
    Aggregation Service-->>Client: 200 OK<br/>{total_balance_usd: 2500.00,<br/>breakdown: [{EUR: 1000}, {USD: 1420}]}
```

---

## 6. Idempotency Key Expiration & Cleanup

```mermaid
sequenceDiagram
    participant Scheduler
    participant Cleanup Service
    participant Oracle DB
    participant Redis

    Note over Scheduler: Cron Job (Every Hour)
    Scheduler->>Cleanup Service: Trigger Cleanup

    Cleanup Service->>Oracle DB: SELECT id, key FROM idempotency_keys<br/>WHERE status='PENDING'<br/>AND created_at < NOW() - INTERVAL '{ttl}'

    loop For Each Expired Key
        Cleanup Service->>Oracle DB: BEGIN TRANSACTION
        Cleanup Service->>Oracle DB: UPDATE idempotency_keys<br/>SET status='EXPIRED'<br/>WHERE id=:id
        Cleanup Service->>Oracle DB: COMMIT

        Note over Cleanup Service,Redis: Remove Lock
        Cleanup Service->>Redis: DEL idempotency:{key}
    end

    Cleanup Service->>Oracle DB: DELETE FROM idempotency_keys<br/>WHERE status='EXPIRED'<br/>AND created_at < NOW() - INTERVAL '30 days'
```

---

## Performance Characteristics

| Flow | Latency (p95) | Throughput | Bottleneck |
|------|---------------|------------|------------|
| **Payment Processing** | < 100ms | 10,000 TPS | Database writes |
| **Cached Balance Query** | < 10ms | 100,000 TPS | Redis throughput |
| **Computed Balance** | < 50ms | 5,000 TPS | Database aggregation |
| **Rollback** | < 200ms | 1,000 TPS | Multi-table updates |
| **Discount Validation** | < 20ms | 50,000 TPS | Redis cache |
| **Wallet Creation** | < 30ms | 10,000 TPS | Database inserts |

---

## Next Steps

- Review [Domain Model](../02-domain-model/entity-relationships.md) for database schema
- See [API Specification](../03-api-specification/payment-apis.md) for endpoint details
- Explore [Idempotency Pattern](../06-design-patterns/idempotency-deduplication.md) for implementation details
