# Database Schema

## Overview

Complete Oracle Database schema for the Wallet Service, including all tables, indexes, constraints, partitioning, and security configurations.

---

## Schema Creation (Flyway Migrations)

### V1__initial_schema.sql

```sql
-- =====================================================
-- WALLET SERVICE - ORACLE DATABASE SCHEMA
-- =====================================================

-- Create sequences
CREATE SEQUENCE business_seq START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE user_seq START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE wallet_seq START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE transaction_seq START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE ledger_entry_seq START WITH 1 INCREMENT BY 1;

-- =====================================================
-- BUSINESS TABLE (PARENT-CHILD STRUCTURE)
-- =====================================================
-- Merged business and business line into single hierarchical table
-- Parent records (parent_id IS NULL) represent businesses
-- Child records (parent_id IS NOT NULL) represent business lines/divisions
CREATE TABLE businesses (
    id RAW(16) DEFAULT SYS_GUID() PRIMARY KEY,
    parent_id RAW(16), -- NULL for parent business, populated for business lines
    name VARCHAR2(255) NOT NULL,
    code VARCHAR2(50), -- Business line code (e.g., ECOMMERCE, CRYPTO) - NULL for parent
    business_type VARCHAR2(50) CHECK (business_type IN ('CORPORATION', 'LLC', 'PARTNERSHIP', 'SOLE_PROPRIETOR')),
    tax_id VARCHAR2(500), -- Encrypted - only for parent businesses
    country CHAR(2), -- ISO 3166-1 alpha-2 - only for parent businesses
    email VARCHAR2(255),
    phone VARCHAR2(500), -- Encrypted
    address_line1 VARCHAR2(500),
    address_line2 VARCHAR2(500),
    city VARCHAR2(100),
    state VARCHAR2(100),
    postal_code VARCHAR2(20),
    description VARCHAR2(500), -- Optional description for business lines
    is_active NUMBER(1) DEFAULT 1 NOT NULL CHECK (is_active IN (0, 1)),
    created_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    updated_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    status VARCHAR2(20) DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE', 'SUSPENDED', 'CLOSED')),
    CONSTRAINT fk_business_parent FOREIGN KEY (parent_id) REFERENCES businesses(id),
    CONSTRAINT uk_business_line_code UNIQUE (parent_id, code),
    CONSTRAINT check_parent_required_fields CHECK (
        (parent_id IS NULL AND business_type IS NOT NULL AND country IS NOT NULL AND email IS NOT NULL) OR
        (parent_id IS NOT NULL AND code IS NOT NULL)
    )
);

CREATE UNIQUE INDEX idx_business_tax_country ON businesses(tax_id, country) WHERE parent_id IS NULL;
CREATE INDEX idx_business_status ON businesses(status);
CREATE INDEX idx_business_created ON businesses(created_at DESC);
CREATE INDEX idx_business_parent ON businesses(parent_id);
CREATE INDEX idx_business_active ON businesses(is_active) WHERE is_active = 1;

COMMENT ON TABLE businesses IS 'Hierarchical business entities with parent-child structure for business lines';
COMMENT ON COLUMN businesses.parent_id IS 'NULL for parent business, populated for business lines/divisions';
COMMENT ON COLUMN businesses.code IS 'Unique code per parent business for business lines (e.g., ECOMMERCE, CRYPTO)';
COMMENT ON COLUMN businesses.tax_id IS 'Encrypted tax identification number - only for parent businesses';
COMMENT ON COLUMN businesses.description IS 'Optional description - primarily for business lines';

-- =====================================================
-- USER TABLE
-- =====================================================
CREATE TABLE users (
    id RAW(16) DEFAULT SYS_GUID() PRIMARY KEY,
    business_id RAW(16),
    email VARCHAR2(255) NOT NULL UNIQUE,
    encrypted_phone VARCHAR2(500),
    full_name VARCHAR2(255) NOT NULL,
    role VARCHAR2(50) DEFAULT 'USER' CHECK (role IN ('USER', 'BUSINESS_ADMIN', 'FINANCE_ADMIN', 'SYSTEM_ADMIN')),
    kyc_status VARCHAR2(20) DEFAULT 'NOT_STARTED' CHECK (kyc_status IN ('NOT_STARTED', 'PENDING', 'APPROVED', 'REJECTED')),
    kyc_verified_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    updated_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    status VARCHAR2(20) DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE', 'SUSPENDED', 'CLOSED')),
    CONSTRAINT fk_user_business FOREIGN KEY (business_id) REFERENCES businesses(id)
);

CREATE INDEX idx_user_business ON users(business_id);
CREATE INDEX idx_user_email ON users(LOWER(email));
CREATE INDEX idx_user_status ON users(status);
CREATE INDEX idx_user_kyc ON users(kyc_status);

COMMENT ON TABLE users IS 'User accounts (B2C and B2B employees)';
COMMENT ON COLUMN users.business_id IS 'NULL for B2C users, populated for B2B (parent business only)';

-- =====================================================
-- WALLET TABLE (ENHANCED - MULTI-BUSINESS + BUSINESS LINE SUPPORT)
-- =====================================================
-- Note: business_id can reference either parent business or business line
-- When referencing business line, it represents line-specific wallet
-- When referencing parent business (where parent_id IS NULL), it represents business-level wallet
CREATE TABLE wallets (
    id RAW(16) DEFAULT SYS_GUID() PRIMARY KEY,
    user_id RAW(16) NOT NULL,
    business_id RAW(16), -- NULL for B2C, references parent business OR business line
    currency CHAR(3) NOT NULL, -- ISO 4217
    wallet_type VARCHAR2(20) DEFAULT 'STANDARD' CHECK (wallet_type IN ('STANDARD', 'SAVINGS', 'BUSINESS', 'ESCROW', 'PERSONAL')),
    reserved_amount NUMBER(19,0) DEFAULT 0 NOT NULL,
    wallet_name VARCHAR2(255), -- User-friendly name (e.g., "E-commerce USD", "Crypto BTC")
    is_active NUMBER(1) DEFAULT 1 NOT NULL CHECK (is_active IN (0, 1)), -- Boolean: 1=active, 0=inactive
    created_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    updated_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    status VARCHAR2(20) DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE', 'FROZEN', 'CLOSED', 'SUSPENDED')),
    version NUMBER DEFAULT 1 NOT NULL, -- Optimistic locking
    CONSTRAINT fk_wallet_user FOREIGN KEY (user_id) REFERENCES users(id),
    CONSTRAINT fk_wallet_business FOREIGN KEY (business_id) REFERENCES businesses(id),
    CONSTRAINT check_reserved_positive CHECK (reserved_amount >= 0)
);

-- UNIQUE CONSTRAINT: Allows multiple wallets per user/currency across business/business lines
CREATE UNIQUE INDEX idx_wallet_user_business_currency ON wallets(
    user_id,
    COALESCE(business_id, RAW '00000000000000000000000000000000'),
    currency,
    wallet_type
);

CREATE INDEX idx_wallet_user ON wallets(user_id);
CREATE INDEX idx_wallet_business ON wallets(business_id);
CREATE INDEX idx_wallet_status ON wallets(status);
CREATE INDEX idx_wallet_currency ON wallets(currency);
CREATE INDEX idx_wallet_active ON wallets(is_active) WHERE is_active = 1;

COMMENT ON TABLE wallets IS 'User wallets with hierarchical business support';
COMMENT ON COLUMN wallets.business_id IS 'NULL for B2C, references parent business or business line from businesses table';
COMMENT ON COLUMN wallets.wallet_name IS 'Optional user-friendly name for easy identification';
COMMENT ON COLUMN wallets.is_active IS 'Active flag: 1=active, 0=inactive (soft delete)';
COMMENT ON COLUMN wallets.reserved_amount IS 'Funds reserved for pending transactions (in minor units)';
COMMENT ON COLUMN wallets.version IS 'Optimistic locking version for concurrent updates';

-- Example Data:
-- First create parent business:
--   INSERT INTO businesses (id, parent_id, name, business_type, country, email)
--   VALUES ('biz-001', NULL, 'Acme Corp', 'CORPORATION', 'US', 'contact@acme.com');
-- Then create business lines as children:
--   INSERT INTO businesses (id, parent_id, name, code) VALUES
--     ('line-ecom', 'biz-001', 'E-commerce', 'ECOMMERCE'),
--     ('line-crypto', 'biz-001', 'Crypto', 'CRYPTO');
--
-- Scenario 1: User Alice works for Acme Corp and TechStart (multi-business)
-- Personal:    (user_id=alice, business_id=NULL, currency=USD, type=PERSONAL)
-- Acme Corp:   (user_id=alice, business_id=biz-001, currency=USD, type=BUSINESS) -- parent business
-- TechStart:   (user_id=alice, business_id=biz-002, currency=USD, type=BUSINESS) -- different parent
--
-- Scenario 2: User John works in business-1 with business lines (intra-business segregation)
-- Personal:       (user_id=john, business_id=NULL, currency=USD, type=PERSONAL)
-- E-commerce:     (user_id=john, business_id=line-ecom, currency=USD, type=BUSINESS) -- business line
-- Crypto USD:     (user_id=john, business_id=line-crypto, currency=USD, type=BUSINESS) -- business line
-- Crypto BTC:     (user_id=john, business_id=line-crypto, currency=BTC, type=BUSINESS) -- same line, diff currency
-- All wallets have completely separate balances

-- =====================================================
-- TRANSACTION TABLE (PARTITIONED BY MONTH)
-- =====================================================
CREATE TABLE transactions (
    id RAW(16) DEFAULT SYS_GUID() PRIMARY KEY,
    user_id RAW(16) NOT NULL,
    business_id RAW(16),
    source_wallet_id RAW(16),
    destination_wallet_id RAW(16),
    amount NUMBER(19,0) NOT NULL,
    currency CHAR(3) NOT NULL,
    payment_type VARCHAR2(20) NOT NULL CHECK (payment_type IN ('PREPAYMENT', 'POSTPAYMENT', 'VALUABLE', 'CREDIT', 'ROLLBACK', 'TRANSFER')),
    status VARCHAR2(20) DEFAULT 'PENDING' CHECK (status IN ('PENDING', 'CONFIRMED', 'COMPLETED', 'FAILED', 'ROLLED_BACK')),
    idempotency_key VARCHAR2(255),
    ref_transaction_id RAW(16), -- For rollbacks
    gateway_transaction_id VARCHAR2(255),
    created_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    updated_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    created_by VARCHAR2(255), -- Actor (user_id or system)
    correlation_id VARCHAR2(100), -- Distributed tracing
    description VARCHAR2(1000),
    version NUMBER DEFAULT 1 NOT NULL,
    CONSTRAINT fk_transaction_user FOREIGN KEY (user_id) REFERENCES users(id),
    CONSTRAINT fk_transaction_business FOREIGN KEY (business_id) REFERENCES businesses(id),
    CONSTRAINT fk_transaction_source_wallet FOREIGN KEY (source_wallet_id) REFERENCES wallets(id),
    CONSTRAINT fk_transaction_dest_wallet FOREIGN KEY (destination_wallet_id) REFERENCES wallets(id),
    CONSTRAINT fk_transaction_ref FOREIGN KEY (ref_transaction_id) REFERENCES transactions(id),
    CONSTRAINT check_amount_positive CHECK (amount > 0)
)
PARTITION BY RANGE (created_at) INTERVAL (NUMTOYMINTERVAL(1, 'MONTH')) (
    PARTITION tx_initial VALUES LESS THAN (TO_DATE('2025-01-01', 'YYYY-MM-DD'))
);

CREATE INDEX idx_transaction_user_created ON transactions(user_id, created_at DESC) LOCAL;
CREATE INDEX idx_transaction_business_created ON transactions(business_id, created_at DESC) LOCAL;
CREATE INDEX idx_transaction_status ON transactions(status) LOCAL;
CREATE INDEX idx_transaction_correlation ON transactions(correlation_id);
CREATE UNIQUE INDEX idx_transaction_idempotency ON transactions(idempotency_key) WHERE idempotency_key IS NOT NULL;
CREATE INDEX idx_transaction_ref ON transactions(ref_transaction_id);

COMMENT ON TABLE transactions IS 'Immutable transaction records (partitioned by month)';
COMMENT ON COLUMN transactions.amount IS 'Transaction amount in minor units (e.g., cents)';

-- =====================================================
-- LEDGER_ENTRY TABLE
-- =====================================================
CREATE TABLE ledger_entries (
    id RAW(16) DEFAULT SYS_GUID() PRIMARY KEY,
    transaction_id RAW(16) NOT NULL,
    wallet_id RAW(16) NOT NULL,
    entry_type VARCHAR2(10) NOT NULL CHECK (entry_type IN ('DEBIT', 'CREDIT')),
    amount NUMBER(19,0) NOT NULL,
    currency CHAR(3) NOT NULL,
    created_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    status VARCHAR2(20) DEFAULT 'CONFIRMED' CHECK (status IN ('RESERVED', 'CONFIRMED', 'CANCELLED')),
    CONSTRAINT fk_ledger_transaction FOREIGN KEY (transaction_id) REFERENCES transactions(id),
    CONSTRAINT fk_ledger_wallet FOREIGN KEY (wallet_id) REFERENCES wallets(id),
    CONSTRAINT check_ledger_amount_positive CHECK (amount > 0)
);

CREATE INDEX idx_ledger_transaction ON ledger_entries(transaction_id);
CREATE INDEX idx_ledger_wallet_status ON ledger_entries(wallet_id, status);
CREATE INDEX idx_ledger_created ON ledger_entries(created_at DESC);

COMMENT ON TABLE ledger_entries IS 'Immutable double-entry ledger records';
COMMENT ON COLUMN ledger_entries.entry_type IS 'DEBIT decreases balance, CREDIT increases balance';
COMMENT ON COLUMN ledger_entries.amount IS 'Always positive; direction from entry_type';

-- =====================================================
-- PAYMENT_METADATA TABLE
-- =====================================================
CREATE TABLE payment_metadata (
    id RAW(16) DEFAULT SYS_GUID() PRIMARY KEY,
    transaction_id RAW(16) NOT NULL,
    key VARCHAR2(100) NOT NULL,
    value CLOB,
    created_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    CONSTRAINT fk_metadata_transaction FOREIGN KEY (transaction_id) REFERENCES transactions(id)
);

CREATE INDEX idx_metadata_transaction ON payment_metadata(transaction_id);
CREATE INDEX idx_metadata_key ON payment_metadata(key);

COMMENT ON TABLE payment_metadata IS 'Flexible key-value metadata for transactions';

-- =====================================================
-- CREDIT_ACCOUNT TABLE
-- =====================================================
CREATE TABLE credit_accounts (
    id RAW(16) DEFAULT SYS_GUID() PRIMARY KEY,
    user_id RAW(16) NOT NULL UNIQUE,
    credit_limit NUMBER(19,0) NOT NULL,
    available_credit NUMBER(19,0) NOT NULL,
    outstanding_balance NUMBER(19,0) DEFAULT 0 NOT NULL,
    created_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    updated_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    status VARCHAR2(20) DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE', 'SUSPENDED', 'CLOSED')),
    version NUMBER DEFAULT 1 NOT NULL,
    CONSTRAINT fk_credit_user FOREIGN KEY (user_id) REFERENCES users(id),
    CONSTRAINT check_credit_available CHECK (available_credit = credit_limit - outstanding_balance),
    CONSTRAINT check_outstanding_positive CHECK (outstanding_balance >= 0)
);

CREATE INDEX idx_credit_user ON credit_accounts(user_id);

COMMENT ON TABLE credit_accounts IS 'User credit accounts for postpayment and credit transactions';

-- =====================================================
-- INSTALLMENT_SCHEDULE TABLE
-- =====================================================
CREATE TABLE installment_schedules (
    id RAW(16) DEFAULT SYS_GUID() PRIMARY KEY,
    transaction_id RAW(16) NOT NULL UNIQUE,
    total_installments NUMBER(3) NOT NULL,
    paid_installments NUMBER(3) DEFAULT 0,
    installment_amount NUMBER(19,0) NOT NULL,
    frequency VARCHAR2(20) DEFAULT 'MONTHLY' CHECK (frequency IN ('WEEKLY', 'BIWEEKLY', 'MONTHLY', 'QUARTERLY')),
    next_due_date DATE NOT NULL,
    created_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    status VARCHAR2(20) DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE', 'COMPLETED', 'DEFAULTED')),
    CONSTRAINT fk_installment_transaction FOREIGN KEY (transaction_id) REFERENCES transactions(id),
    CONSTRAINT check_installments_valid CHECK (paid_installments <= total_installments)
);

CREATE INDEX idx_installment_next_due ON installment_schedules(next_due_date, status);

COMMENT ON TABLE installment_schedules IS 'Installment schedules for CREDIT payment type';

-- =====================================================
-- IDEMPOTENCY_KEY TABLE
-- =====================================================
CREATE TABLE idempotency_keys (
    id RAW(16) DEFAULT SYS_GUID() PRIMARY KEY,
    key VARCHAR2(255) NOT NULL,
    user_id RAW(16) NOT NULL,
    endpoint VARCHAR2(255) NOT NULL,
    status VARCHAR2(20) DEFAULT 'PENDING' CHECK (status IN ('PENDING', 'CONFIRMED', 'EXPIRED')),
    cached_response CLOB,
    created_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    ttl_minutes NUMBER(5) NOT NULL,
    CONSTRAINT fk_idempotency_user FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE UNIQUE INDEX idx_idempotency_unique ON idempotency_keys(key, user_id, endpoint);
CREATE INDEX idx_idempotency_expires ON idempotency_keys(expires_at) WHERE status = 'PENDING';
CREATE INDEX idx_idempotency_status ON idempotency_keys(status, created_at);

COMMENT ON TABLE idempotency_keys IS 'Idempotency enforcement with TTL-based expiration';
COMMENT ON COLUMN idempotency_keys.cached_response IS 'Serialized JSON response for idempotent replay';

-- =====================================================
-- AUDIT_LOG TABLE
-- =====================================================
CREATE TABLE audit_logs (
    id RAW(16) DEFAULT SYS_GUID() PRIMARY KEY,
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

CREATE INDEX idx_audit_entity ON audit_logs(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_logs(user_id, created_at DESC);
CREATE INDEX idx_audit_created ON audit_logs(created_at DESC);
CREATE INDEX idx_audit_correlation ON audit_logs(correlation_id);

COMMENT ON TABLE audit_logs IS 'Immutable audit trail for all operations';

-- =====================================================
-- MATERIALIZED VIEWS FOR AGGREGATION
-- =====================================================

-- User balance summary (refreshed on schedule or on-demand)
CREATE MATERIALIZED VIEW user_balance_summary
BUILD IMMEDIATE
REFRESH COMPLETE ON DEMAND
AS
SELECT
    w.user_id,
    w.currency,
    SUM(CASE WHEN le.entry_type = 'CREDIT' THEN le.amount ELSE -le.amount END) as balance,
    COUNT(DISTINCT le.transaction_id) as transaction_count,
    MAX(le.created_at) as last_transaction_at
FROM wallets w
LEFT JOIN ledger_entries le ON w.id = le.wallet_id AND le.status = 'CONFIRMED'
GROUP BY w.user_id, w.currency;

CREATE INDEX idx_mv_user_balance ON user_balance_summary(user_id, currency);

-- Business balance summary
CREATE MATERIALIZED VIEW business_balance_summary
BUILD IMMEDIATE
REFRESH COMPLETE ON DEMAND
AS
SELECT
    b.id as business_id,
    w.currency,
    SUM(CASE WHEN le.entry_type = 'CREDIT' THEN le.amount ELSE -le.amount END) as balance,
    COUNT(DISTINCT le.transaction_id) as transaction_count
FROM businesses b
JOIN wallets w ON b.id = w.business_id
LEFT JOIN ledger_entries le ON w.id = le.wallet_id AND le.status = 'CONFIRMED'
GROUP BY b.id, w.currency;

CREATE INDEX idx_mv_business_balance ON business_balance_summary(business_id, currency);
```

---

### V2__encryption_setup.sql

```sql
-- =====================================================
-- TRANSPARENT DATA ENCRYPTION (TDE) SETUP
-- =====================================================

-- Enable TDE for sensitive columns
ALTER TABLE businesses MODIFY (tax_id ENCRYPT USING 'AES256');
ALTER TABLE businesses MODIFY (phone ENCRYPT USING 'AES256');
ALTER TABLE users MODIFY (encrypted_phone ENCRYPT USING 'AES256');

-- Create encrypted tablespace for sensitive data (optional)
CREATE TABLESPACE encrypted_ts
DATAFILE 'encrypted_ts.dbf' SIZE 500M
ENCRYPTION USING 'AES256'
DEFAULT STORAGE(ENCRYPT);
```

---

### V3__add_indexes_performance.sql

```sql
-- =====================================================
-- PERFORMANCE OPTIMIZATION INDEXES
-- =====================================================

-- Bitmap indexes for low-cardinality columns (good for analytics)
CREATE BITMAP INDEX idx_transaction_payment_type ON transactions(payment_type) LOCAL;
CREATE BITMAP INDEX idx_transaction_status_bitmap ON transactions(status) LOCAL;

-- Covering index for balance queries
CREATE INDEX idx_ledger_balance_covering ON ledger_entries(wallet_id, status, entry_type, amount);

-- Compound index for common query patterns
CREATE INDEX idx_transaction_user_status_created ON transactions(user_id, status, created_at DESC) LOCAL;
CREATE INDEX idx_transaction_business_status_created ON transactions(business_id, status, created_at DESC) LOCAL;

-- Function-based index for case-insensitive email search
CREATE INDEX idx_user_email_lower ON users(LOWER(email));
```

---

## Constraints Summary

### Primary Keys
- All tables use `RAW(16)` UUIDs as primary keys
- Generated via `SYS_GUID()` for global uniqueness

### Foreign Keys
- Cascading deletes disabled (preserve audit trail)
- Referential integrity enforced at database level
- Indexed for join performance

### Check Constraints
```sql
-- Financial integrity
CHECK (amount > 0)
CHECK (reserved_amount >= 0)
CHECK (available_credit = credit_limit - outstanding_balance)

-- Enumeration constraints
CHECK (status IN ('ACTIVE', 'SUSPENDED', 'CLOSED'))
CHECK (payment_type IN ('PREPAYMENT', 'POSTPAYMENT', ...))
CHECK (entry_type IN ('DEBIT', 'CREDIT'))

-- Business rules
CHECK (paid_installments <= total_installments)
CHECK (parent_id IS NULL AND business_type IS NOT NULL OR parent_id IS NOT NULL AND code IS NOT NULL)
```

### Unique Constraints
```sql
-- Prevent duplicates
UNIQUE (email)  -- users.email
UNIQUE (parent_id, code)  -- businesses (business line code unique per parent)
UNIQUE (user_id, business_id, currency, wallet_type)  -- wallets
UNIQUE (key, user_id, endpoint)  -- idempotency_keys
```

---

## Partitioning Details

### Transactions Table (Interval Partitioning)
```sql
-- Auto-creates monthly partitions
PARTITION BY RANGE (created_at)
INTERVAL (NUMTOYMINTERVAL(1, 'MONTH'))
```

**Benefits**:
- Queries within date range only scan relevant partitions
- Old partitions can be archived to cold storage
- Maintenance operations (ANALYZE, INDEX REBUILD) on individual partitions

**Partition Management**:
```sql
-- View partitions
SELECT table_name, partition_name, high_value
FROM user_tab_partitions
WHERE table_name = 'TRANSACTIONS'
ORDER BY partition_position DESC;

-- Archive old partition (example)
ALTER TABLE transactions MOVE PARTITION tx_2024_01 TABLESPACE archive_ts;
```

---

## Sequences

```sql
-- Sequences for numeric IDs (if needed alongside UUIDs)
CREATE SEQUENCE business_seq START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE user_seq START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE transaction_seq START WITH 1 INCREMENT BY 1;
```

---

## Triggers

### Updated_At Trigger
```sql
CREATE OR REPLACE TRIGGER trg_businesses_updated
BEFORE UPDATE ON businesses
FOR EACH ROW
BEGIN
    :NEW.updated_at := SYSTIMESTAMP;
END;
/

CREATE OR REPLACE TRIGGER trg_users_updated
BEFORE UPDATE ON users
FOR EACH ROW
BEGIN
    :NEW.updated_at := SYSTIMESTAMP;
END;
/

CREATE OR REPLACE TRIGGER trg_wallets_updated
BEFORE UPDATE ON wallets
FOR EACH ROW
BEGIN
    :NEW.updated_at := SYSTIMESTAMP;
END;
/

CREATE OR REPLACE TRIGGER trg_transactions_updated
BEFORE UPDATE ON transactions
FOR EACH ROW
BEGIN
    :NEW.updated_at := SYSTIMESTAMP;
END;
/
```

### Audit Trigger
```sql
CREATE OR REPLACE TRIGGER trg_transaction_audit
AFTER INSERT OR UPDATE OR DELETE ON transactions
FOR EACH ROW
DECLARE
    v_action VARCHAR2(10);
BEGIN
    IF INSERTING THEN
        v_action := 'INSERT';
        INSERT INTO audit_logs (event_type, entity_type, entity_id, action, new_value)
        VALUES ('TRANSACTION_CREATED', 'TRANSACTION', :NEW.id, v_action,
                '{"amount":' || :NEW.amount || ',"status":"' || :NEW.status || '"}');
    ELSIF UPDATING THEN
        v_action := 'UPDATE';
        INSERT INTO audit_logs (event_type, entity_type, entity_id, action, old_value, new_value)
        VALUES ('TRANSACTION_UPDATED', 'TRANSACTION', :NEW.id, v_action,
                '{"status":"' || :OLD.status || '"}',
                '{"status":"' || :NEW.status || '"}');
    ELSIF DELETING THEN
        v_action := 'DELETE';
        INSERT INTO audit_logs (event_type, entity_type, entity_id, action, old_value)
        VALUES ('TRANSACTION_DELETED', 'TRANSACTION', :OLD.id, v_action,
                '{"amount":' || :OLD.amount || '}');
    END IF;
END;
/
```

---

## Views

### Active Wallets with Computed Balance
```sql
CREATE OR REPLACE VIEW v_wallet_balances AS
SELECT
    w.id as wallet_id,
    w.user_id,
    w.currency,
    w.wallet_type,
    w.reserved_amount,
    COALESCE(SUM(CASE
        WHEN le.entry_type = 'CREDIT' THEN le.amount
        WHEN le.entry_type = 'DEBIT' THEN -le.amount
        ELSE 0
    END), 0) as computed_balance,
    COALESCE(SUM(CASE
        WHEN le.entry_type = 'CREDIT' THEN le.amount
        WHEN le.entry_type = 'DEBIT' THEN -le.amount
        ELSE 0
    END), 0) - w.reserved_amount as available_balance,
    w.status
FROM wallets w
LEFT JOIN ledger_entries le ON w.id = le.wallet_id AND le.status = 'CONFIRMED'
WHERE w.status = 'ACTIVE'
GROUP BY w.id, w.user_id, w.currency, w.wallet_type, w.reserved_amount, w.status;
```

---

## Security

### Row-Level Security (VPD)
```sql
-- Example: Users can only see their own wallets
CREATE OR REPLACE FUNCTION wallet_security_policy(
    schema_var IN VARCHAR2,
    table_var IN VARCHAR2
)
RETURN VARCHAR2
IS
    v_predicate VARCHAR2(400);
BEGIN
    v_predicate := 'user_id = SYS_CONTEXT(''APP_CTX'', ''USER_ID'')';
    RETURN v_predicate;
END;
/

BEGIN
    DBMS_RLS.ADD_POLICY(
        object_schema   => 'WALLET_SCHEMA',
        object_name     => 'WALLETS',
        policy_name     => 'wallet_user_policy',
        function_schema => 'WALLET_SCHEMA',
        policy_function => 'wallet_security_policy',
        statement_types => 'SELECT, UPDATE, DELETE'
    );
END;
/
```

---

## Next Steps

- Review [Ledger Model](./ledger-model.md) for accounting rules
- See [Payment Types](./payment-types.md) for type-specific logic
- Explore [Idempotency Pattern](../06-design-patterns/idempotency-deduplication.md) for duplicate protection
