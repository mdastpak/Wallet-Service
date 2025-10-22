# Wallet Service - Mermaid Diagrams Collection

This file contains all Mermaid diagrams extracted from the documentation for easy copy-paste into [Mermaid Live Editor](https://mermaid.live)

---

## Table of Contents

1. [Architecture Diagrams](#architecture-diagrams)
2. [Entity Relationships](#entity-relationships)
3. [User Journeys](#user-journeys)
4. [Workflows](#workflows)
5. [Sequence Diagrams](#sequence-diagrams)
6. [State Machines](#state-machines)

---

## Architecture Diagrams

### 1. High-Level System Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        WEB[Web Application]
        MOBILE[Mobile Apps]
        B2B[B2B Partners]
    end

    subgraph "API Gateway Layer"
        APIGW[API Gateway<br/>OAuth 2.0 + mTLS]
    end

    subgraph "Application Layer"
        WALLET[Wallet Service]
        PAYMENT[Payment Service]
        AGGREGATE[Aggregation Service]
        AUTH[Auth Service]
    end

    subgraph "Infrastructure Layer"
        REDIS[(Redis Cluster<br/>Cache + Locks)]
        KAFKA[Kafka<br/>Event Stream]
        ORACLE[(Oracle DB<br/>ACID Store)]
    end

    subgraph "External Systems"
        GATEWAY[Payment Gateway]
        KMS[KMS<br/>Key Management]
    end

    WEB --> APIGW
    MOBILE --> APIGW
    B2B --> APIGW

    APIGW --> WALLET
    APIGW --> PAYMENT
    APIGW --> AGGREGATE
    APIGW --> AUTH

    WALLET --> REDIS
    WALLET --> ORACLE
    WALLET --> KAFKA

    PAYMENT --> REDIS
    PAYMENT --> ORACLE
    PAYMENT --> KAFKA
    PAYMENT --> GATEWAY

    AGGREGATE --> REDIS
    AGGREGATE --> ORACLE

    AUTH --> REDIS

    WALLET -.Encryption.-> KMS
    PAYMENT -.Encryption.-> KMS
```

### 2. Multi-Business Wallet Hierarchy

```mermaid
graph TB
    USER["User: Alice<br/>Email: alice at example.com<br/>Works for multiple businesses"]

    PERSONAL[Personal Context<br/>business_id = NULL]
    ACME[Acme Corp Context<br/>business_id = acme-123]
    TECH[TechStart Context<br/>business_id = tech-456]

    USER --> PERSONAL
    USER --> ACME
    USER --> TECH

    PERSONAL --> P1["USD Personal Wallet<br/>Balance: $5,000<br/>Personal Checking"]
    PERSONAL --> P2["EUR Personal Wallet<br/>Balance: 3000 EUR<br/>Personal Euro"]
    PERSONAL --> P3["BTC Personal Wallet<br/>Balance: 0.5 BTC<br/>Personal Bitcoin"]

    ACME --> A1["USD Business Wallet<br/>Balance: $10,000<br/>Acme Corp USD"]
    ACME --> A2["EUR Business Wallet<br/>Balance: 7500 EUR<br/>Acme Corp EUR"]

    TECH --> T1["USD Business Wallet<br/>Balance: $2,500<br/>TechStart USD"]
    TECH --> T2["BTC Business Wallet<br/>Balance: 0.1 BTC<br/>TechStart BTC"]

    style PERSONAL fill:#fff4e1
    style ACME fill:#e1f5ff
    style TECH fill:#e8f4f8

```

### 3. Kubernetes Deployment Architecture

```mermaid
graph TB
    subgraph "Availability Zone 1"
        subgraph "Control Plane AZ1"
            CP1[Control Plane Node 1]
        end

        subgraph "Worker Nodes AZ1"
            W1[Worker Node 1]
            W2[Worker Node 2]

            W1 --> POD1A[Wallet Service Pod 1]
            W1 --> POD1B[Payment Service Pod 1]
            W2 --> POD1C[Aggregation Service Pod 1]
            W2 --> POD1D[Redis Pod 1]
        end

        subgraph "Data AZ1"
            DB1[(Oracle RAC Node 1)]
            KAFKA1[Kafka Broker 1]
        end
    end

    subgraph "Availability Zone 2"
        subgraph "Control Plane AZ2"
            CP2[Control Plane Node 2]
        end

        subgraph "Worker Nodes AZ2"
            W3[Worker Node 3]
            W4[Worker Node 4]

            W3 --> POD2A[Wallet Service Pod 2]
            W3 --> POD2B[Payment Service Pod 2]
            W4 --> POD2C[Aggregation Service Pod 2]
            W4 --> POD2D[Redis Pod 2]
        end

        subgraph "Data AZ2"
            DB2[(Oracle RAC Node 2)]
            KAFKA2[Kafka Broker 2]
        end
    end

    subgraph "Availability Zone 3"
        subgraph "Control Plane AZ3"
            CP3[Control Plane Node 3]
        end

        subgraph "Worker Nodes AZ3"
            W5[Worker Node 5]

            W5 --> POD3A[Wallet Service Pod 3]
            W5 --> POD3B[Redis Pod 3]
        end

        subgraph "Data AZ3"
            KAFKA3[Kafka Broker 3]
        end
    end

    subgraph "Load Balancer Layer"
        LB[External Load Balancer]
        INGRESS[NGINX Ingress Controller]
    end

    LB --> INGRESS
    INGRESS --> POD1A
    INGRESS --> POD1B
    INGRESS --> POD2A
    INGRESS --> POD2B
    INGRESS --> POD3A

    DB1 <-.Oracle RAC Interconnect.-> DB2
    KAFKA1 <-.Kafka Replication.-> KAFKA2
    KAFKA2 <-.Kafka Replication.-> KAFKA3
    POD1D <-.Redis Cluster.-> POD2D
    POD2D <-.Redis Cluster.-> POD3B

    style LB fill:#d4edda
    style CP1 fill:#e1f5ff
    style CP2 fill:#e1f5ff
    style CP3 fill:#e1f5ff
```

---

## Entity Relationships

### Complete ER Diagram (Hierarchical Business Structure)

```mermaid
erDiagram
    BUSINESS ||--o{ BUSINESS : "has_children"
    BUSINESS ||--o{ USER : "employs"
    BUSINESS ||--o{ WALLET : "contains"
    USER ||--o{ WALLET : "owns"
    WALLET ||--o{ LEDGER_ENTRY : "records"
    TRANSACTION ||--o{ LEDGER_ENTRY : "contains"
    USER ||--o{ TRANSACTION : "initiates"
    BUSINESS ||--o{ TRANSACTION : "processes"
    TRANSACTION ||--o| TRANSACTION : "rollback_of"
    TRANSACTION ||--o{ PAYMENT_METADATA : "has"
    USER ||--o{ CREDIT_ACCOUNT : "holds"
    TRANSACTION ||--o{ INSTALLMENT_SCHEDULE : "has"
    IDEMPOTENCY_KEY ||--|| TRANSACTION : "ensures_uniqueness"

    BUSINESS {
        uuid id PK
        uuid parent_id FK
        string name
        string code
        string business_type
        string tax_id
        string country
        string email
        string phone
        string description
        boolean is_active
        timestamp created_at
        timestamp updated_at
        string status
    }

    USER {
        uuid id PK
        uuid business_id FK
        string email
        string encrypted_phone
        string full_name
        string role
        string kyc_status
        timestamp kyc_verified_at
        timestamp created_at
        timestamp updated_at
        string status
    }

    WALLET {
        uuid id PK
        uuid user_id FK
        uuid business_id FK
        string currency
        string wallet_type
        string wallet_name
        bigint reserved_amount
        boolean is_active
        timestamp created_at
        timestamp updated_at
        string status
        int version
    }

    TRANSACTION {
        uuid id PK
        uuid user_id FK
        uuid business_id FK
        uuid source_wallet_id FK
        uuid destination_wallet_id FK
        bigint amount
        string currency
        string payment_type
        string status
        string idempotency_key
        uuid ref_transaction_id FK
        string gateway_transaction_id
        string correlation_id
        timestamp created_at
        timestamp updated_at
        int version
    }

    LEDGER_ENTRY {
        uuid id PK
        uuid transaction_id FK
        uuid wallet_id FK
        string entry_type
        bigint amount
        string currency
        timestamp created_at
        string status
    }

    PAYMENT_METADATA {
        uuid id PK
        uuid transaction_id FK
        string key
        string value
        timestamp created_at
    }

    CREDIT_ACCOUNT {
        uuid id PK
        uuid user_id FK
        bigint credit_limit
        bigint outstanding_balance
        decimal interest_rate
        int grace_period_days
        timestamp created_at
        timestamp updated_at
        string status
    }

    INSTALLMENT_SCHEDULE {
        uuid id PK
        uuid transaction_id FK
        int installment_number
        bigint installment_amount
        timestamp due_date
        timestamp paid_at
        string status
        timestamp created_at
    }

    IDEMPOTENCY_KEY {
        uuid id PK
        string key
        uuid user_id FK
        string endpoint
        string status
        text cached_response
        timestamp created_at
        timestamp expires_at
        int ttl_minutes
    }
```

---

## User Journeys

### Consumer Payment Journey

```mermaid
journey
    title Consumer Payment Journey - PREPAYMENT

    section Registration & Setup
        Sign up with email: 5: Consumer
        Complete KYC verification: 3: Consumer
        Create USD wallet: 5: Consumer

    section Browse & Select
        Browse products: 5: Consumer
        Add items to cart: 5: Consumer
        View cart total: 5: Consumer

    section Payment
        Initiate payment: 5: Consumer
        System reserves funds: 3: System
        Receive payment confirmation: 5: Consumer

    section Fulfillment
        Merchant ships product: 3: Merchant
        System captures payment: 5: System
        Receive completion notification: 5: Consumer

    section Post-Transaction
        View transaction history: 5: Consumer
        Check updated balance: 5: Consumer
```

---

## Workflows

### Payment Type Decision Tree

```mermaid
flowchart TD
    START([Payment Request]) --> CHECK_TYPE{Payment Type?}

    CHECK_TYPE -->|PREPAYMENT| PRE_START[Prepayment Flow]
    CHECK_TYPE -->|POSTPAYMENT| POST_START[Postpayment Flow]
    CHECK_TYPE -->|VALUABLE| VAL_START[Valuable Flow]
    CHECK_TYPE -->|CREDIT| CREDIT_START[Credit Flow]

    PRE_START --> PRE_RESERVE[Reserve Funds Immediately]
    PRE_RESERVE --> PRE_LEDGER[Create RESERVED Ledger Entry]
    PRE_LEDGER --> PRE_WAIT[Wait for Capture or Cancel]
    PRE_WAIT --> PRE_DECISION{Capture or<br/>Cancel?}
    PRE_DECISION -->|Capture| PRE_CONFIRM[Confirm Ledger Entry]
    PRE_DECISION -->|Cancel| PRE_CANCEL[Release Reservation]
    PRE_CONFIRM --> END([Success])
    PRE_CANCEL --> END

    POST_START --> POST_CHECK_CREDIT{Credit<br/>Available?}
    POST_CHECK_CREDIT -->|No| POST_FAIL[Reject - Credit Limit]
    POST_CHECK_CREDIT -->|Yes| POST_INVOICE[Create Invoice]
    POST_INVOICE --> POST_UPDATE[Update Credit Account]
    POST_UPDATE --> POST_WAIT[Wait for Settlement]
    POST_WAIT --> POST_SETTLE[Settle Invoice]
    POST_SETTLE --> POST_LEDGER[Create Ledger Entries]
    POST_LEDGER --> END
    POST_FAIL --> END

    VAL_START --> VAL_THRESHOLD{Amount ><br/>$10,000?}
    VAL_THRESHOLD -->|No| VAL_FAIL[Not Valuable Transaction]
    VAL_THRESHOLD -->|Yes| VAL_KYC{KYC<br/>Approved?}
    VAL_KYC -->|No| VAL_KYC_FAIL[Require KYC]
    VAL_KYC -->|Yes| VAL_2FA[Send 2FA Code]
    VAL_2FA --> VAL_VERIFY{2FA<br/>Valid?}
    VAL_VERIFY -->|No| VAL_2FA_FAIL[2FA Failed]
    VAL_VERIFY -->|Yes| VAL_RISK[Evaluate Risk Score]
    VAL_RISK --> VAL_RISK_CHECK{Risk<br/>Acceptable?}
    VAL_RISK_CHECK -->|High Risk| VAL_REVIEW[Manual Review Required]
    VAL_RISK_CHECK -->|Low Risk| VAL_PROCESS[Process Payment]
    VAL_PROCESS --> END
    VAL_REVIEW --> END
    VAL_FAIL --> END
    VAL_KYC_FAIL --> END
    VAL_2FA_FAIL --> END

    CREDIT_START --> CREDIT_PLAN[Create Installment Schedule]
    CREDIT_PLAN --> CREDIT_CALC[Calculate Monthly Payments]
    CREDIT_CALC --> CREDIT_SCHEDULE[Insert Schedule Details]
    CREDIT_SCHEDULE --> CREDIT_FIRST[Process First Installment]
    CREDIT_FIRST --> CREDIT_CRON[Schedule Monthly Job]
    CREDIT_CRON --> END

    style START fill:#d4edda
    style END fill:#d4edda
    style POST_FAIL fill:#f8d7da
    style VAL_FAIL fill:#f8d7da
    style VAL_KYC_FAIL fill:#f8d7da
    style VAL_2FA_FAIL fill:#f8d7da
```

---

## State Machines

### Transaction Lifecycle

```mermaid
stateDiagram-v2
    [*] --> PENDING: Payment Initiated

    PENDING --> CONFIRMED: Payment Gateway Success
    PENDING --> FAILED: Payment Gateway Failed
    PENDING --> EXPIRED: Timeout (Prepayment)
    PENDING --> PENDING_REVIEW: High Risk (Valuable)

    PENDING_REVIEW --> CONFIRMED: Admin Approved
    PENDING_REVIEW --> REJECTED: Admin Rejected

    CONFIRMED --> COMPLETED: Funds Settled
    CONFIRMED --> ROLLED_BACK: Admin Rollback

    EXPIRED --> [*]
    FAILED --> [*]
    REJECTED --> [*]
    COMPLETED --> [*]
    ROLLED_BACK --> [*]

    note right of PENDING
        Initial state for all
        payment types
    end note

    note right of PENDING_REVIEW
        Only for VALUABLE
        transactions with
        high risk score
    end note

    note right of ROLLED_BACK
        Compensation transaction
        created, funds restored
    end note
```

### Wallet Status State Machine

```mermaid
stateDiagram-v2
    [*] --> PENDING: Create Wallet Request

    PENDING --> ACTIVE: Verification Complete
    PENDING --> REJECTED: Verification Failed

    ACTIVE --> FROZEN: Admin/System Freeze
    ACTIVE --> SUSPENDED: Suspicious Activity
    ACTIVE --> CLOSED: User Closes Wallet

    FROZEN --> ACTIVE: Admin Unfreezes
    FROZEN --> CLOSED: Admin Closes

    SUSPENDED --> ACTIVE: Issue Resolved
    SUSPENDED --> CLOSED: Permanent Suspension

    REJECTED --> [*]
    CLOSED --> [*]

    note right of FROZEN
        Funds locked
        No transactions allowed
        Manual intervention required
    end note

    note right of SUSPENDED
        Temporary hold
        Pending investigation
        Can be restored
    end note
```

---

## CQRS Pattern

### Read/Write Segregation

```mermaid
graph TB
    subgraph CMD["Command Side (Write Model)"]
        API_WRITE[POST /v1/payments]
        PAYMENT_CMD[Payment Command Handler]
        ORACLE[(Oracle DB<br/>Source of Truth)]
        KAFKA_WRITE[Kafka: TransactionCompleted]

        API_WRITE --> PAYMENT_CMD
        PAYMENT_CMD --> ORACLE
        PAYMENT_CMD --> KAFKA_WRITE
    end

    subgraph QRY["Query Side (Read Model)"]
        API_READ[GET /v1/wallets/id/balance]
        BALANCE_QUERY[Balance Query Handler]
        REDIS[(Redis Cache<br/>Materialized View)]

        API_READ --> BALANCE_QUERY
        BALANCE_QUERY --> REDIS
        BALANCE_QUERY -.Cache Miss.-> ORACLE
    end

    subgraph EVT["Event Processing"]
        AGGREGATION_CONSUMER[Aggregation Consumer]
        KAFKA_WRITE --> AGGREGATION_CONSUMER
        AGGREGATION_CONSUMER --> REDIS
    end

    style ORACLE fill:#fff4e1
    style REDIS fill:#e1f5ff
    style KAFKA_WRITE fill:#e8f4f8
```

---

## How to Use These Diagrams

### On Mermaid Live Editor

1. **Go to**: <https://mermaid.live>
2. **Copy** any diagram code from above (including the ` ```mermaid ` and ` ``` ` markers)
3. **Paste** into the left editor panel
4. The diagram will render on the right
5. **Export** as PNG, SVG, or share via URL

### Example

Copy this entire block:

```mermaid
graph TB
    USER[User] --> WALLET[Wallet]
    WALLET --> BALANCE[Balance: $1000]
```

Paste into <https://mermaid.live> and it will render!

---

## File Locations

All these diagrams are embedded in:

- `docs/PRODUCT_DESIGN.md` - Main design document
- `docs/01-architecture/data-flows.md` - Sequence diagrams
- `docs/02-domain-model/multi-business-wallets.md` - Multi-business diagrams
- `docs/02-domain-model/entity-relationships.md` - ER diagram

---

**Total Diagrams**: 15+ ready-to-use Mermaid diagrams
**Status**: ✅ Ready for visualization
**Tools**: Compatible with Mermaid Live Editor, GitHub, GitLab, Confluence, etc.
