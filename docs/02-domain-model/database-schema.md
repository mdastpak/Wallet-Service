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
-- BUSINESS TABLE
-- =====================================================
CREATE TABLE businesses (
    id RAW(16) DEFAULT SYS_GUID() PRIMARY KEY,
    name VARCHAR2(255) NOT NULL,
    business_type VARCHAR2(50) NOT NULL CHECK (business_type IN ('CORPORATION', 'LLC', 'PARTNERSHIP', 'SOLE_PROPRIETOR')),
    tax_id VARCHAR2(500), -- Encrypted
    country CHAR(2) NOT NULL, -- ISO 3166-1 alpha-2
    email VARCHAR2(255) NOT NULL,
    phone VARCHAR2(500), -- Encrypted
    address_line1 VARCHAR2(500),
    address_line2 VARCHAR2(500),
    city VARCHAR2(100),
    state VARCHAR2(100),
    postal_code VARCHAR2(20),
    created_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    updated_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    status VARCHAR2(20) DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE', 'SUSPENDED', 'CLOSED'))
);

CREATE UNIQUE INDEX idx_business_tax_country ON businesses(tax_id, country);
CREATE INDEX idx_business_status ON businesses(status);
CREATE INDEX idx_business_created ON businesses(created_at DESC);

COMMENT ON TABLE businesses IS 'Business entities for B2B operations';
COMMENT ON COLUMN businesses.tax_id IS 'Encrypted tax identification number';

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
COMMENT ON COLUMN users.business_id IS 'NULL for B2C users, populated for B2B';

-- =====================================================
-- BUSINESS LINES TABLE (NEW - FOR INTRA-BUSINESS SEGREGATION)
-- =====================================================
CREATE TABLE business_lines (
    id RAW(16) DEFAULT SYS_GUID() PRIMARY KEY,
    business_id RAW(16) NOT NULL,
    name VARCHAR2(100) NOT NULL,
    code VARCHAR2(50) NOT NULL,
    description VARCHAR2(500),
    is_active NUMBER(1) DEFAULT 1 NOT NULL CHECK (is_active IN (0, 1)),
    created_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    updated_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    CONSTRAINT fk_business_line_business FOREIGN KEY (business_id) REFERENCES businesses(id),
    CONSTRAINT uk_business_line_code UNIQUE (business_id, code)
);

CREATE INDEX idx_business_line_business ON business_lines(business_id);
CREATE INDEX idx_business_line_active ON business_lines(is_active) WHERE is_active = 1;

COMMENT ON TABLE business_lines IS 'Business lines/departments within a business (e.g., E-commerce, Crypto, Marketing divisions)';
COMMENT ON COLUMN business_lines.code IS 'Unique code per business (e.g., ECOMMERCE, CRYPTO, MARKETING)';

-- Example Data:
-- INSERT INTO business_lines (id, business_id, name, code) VALUES
--   ('line-001', 'biz-001', 'E-commerce', 'ECOMMERCE'),
--   ('line-002', 'biz-001', 'Crypto', 'CRYPTO'),
--   ('line-003', 'biz-001', 'Marketing', 'MARKETING');

-- =====================================================
-- WALLET TABLE (ENHANCED - MULTI-BUSINESS + BUSINESS LINE SUPPORT)
-- =====================================================
CREATE TABLE wallets (
    id RAW(16) DEFAULT SYS_GUID() PRIMARY KEY,
    user_id RAW(16) NOT NULL,
    business_id RAW(16), -- NULL for personal B2C wallets
    business_line_id RAW(16), -- NULL for business-level or personal wallets
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
    CONSTRAINT fk_wallet_business_line FOREIGN KEY (business_line_id) REFERENCES business_lines(id),
    CONSTRAINT check_reserved_positive CHECK (reserved_amount >= 0),
    CONSTRAINT check_business_line_requires_business CHECK (
        business_line_id IS NULL OR business_id IS NOT NULL
    )
);

-- ENHANCED UNIQUE CONSTRAINT: Allows multiple wallets per user/currency across business lines
CREATE UNIQUE INDEX idx_wallet_user_business_line_currency ON wallets(
    user_id,
    COALESCE(business_id, RAW '00000000000000000000000000000000'),
    COALESCE(business_line_id, RAW '00000000000000000000000000000000'),
    currency,
    wallet_type
);

CREATE INDEX idx_wallet_user ON wallets(user_id);
CREATE INDEX idx_wallet_business ON wallets(business_id);
CREATE INDEX idx_wallet_business_line ON wallets(business_line_id);
CREATE INDEX idx_wallet_status ON wallets(status);
CREATE INDEX idx_wallet_currency ON wallets(currency);
CREATE INDEX idx_wallet_active ON wallets(is_active) WHERE is_active = 1;

COMMENT ON TABLE wallets IS 'User wallets with business and business line segregation support';
COMMENT ON COLUMN wallets.business_id IS 'NULL for personal B2C wallets, populated for business-specific wallets';
COMMENT ON COLUMN wallets.business_line_id IS 'NULL for business-level or personal wallets, populated for business line-specific wallets (e.g., E-commerce, Crypto divisions)';
COMMENT ON COLUMN wallets.wallet_name IS 'Optional user-friendly name for easy identification';
COMMENT ON COLUMN wallets.is_active IS 'Active flag: 1=active, 0=inactive (soft delete)';
COMMENT ON COLUMN wallets.reserved_amount IS 'Funds reserved for pending transactions (in minor units)';
COMMENT ON COLUMN wallets.version IS 'Optimistic locking version for concurrent updates';

-- Example Data:
-- Scenario 1: User Alice works for Acme Corp and TechStart (multi-business)
-- Personal:    (user_id=alice, business_id=NULL,  business_line_id=NULL, currency=USD, type=PERSONAL)
-- Acme Corp:   (user_id=alice, business_id=acme,  business_line_id=NULL, currency=USD, type=BUSINESS)
-- TechStart:   (user_id=alice, business_id=tech,  business_line_id=NULL, currency=USD, type=BUSINESS)
--
-- Scenario 2: User John works in business-1 with 3 business lines (intra-business segregation)
-- Personal:       (user_id=john, business_id=NULL,     business_line_id=NULL,      currency=USD, type=PERSONAL)
-- E-commerce:     (user_id=john, business_id=biz-001,  business_line_id=line-ecom, currency=USD, type=BUSINESS)
-- Crypto:         (user_id=john, business_id=biz-001,  business_line_id=line-crypto, currency=USD, type=BUSINESS)
-- Marketing:      (user_id=john, business_id=biz-001,  business_line_id=line-mkt,  currency=USD, type=BUSINESS)
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
    discount_code_id RAW(16),
    discount_amount NUMBER(19,0) DEFAULT 0,
    final_amount NUMBER(19,0) NOT NULL,
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
    CONSTRAINT check_amount_positive CHECK (amount > 0),
    CONSTRAINT check_final_amount CHECK (final_amount = amount - discount_amount)
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
COMMENT ON COLUMN transactions.final_amount IS 'Amount after discount (computed column check)';

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
-- DISCOUNT_CODE TABLE (ENHANCED)
-- =====================================================
CREATE TABLE discount_codes (
    id RAW(16) DEFAULT SYS_GUID() PRIMARY KEY,
    code VARCHAR2(50) NOT NULL UNIQUE,
    discount_type VARCHAR2(20) NOT NULL CHECK (discount_type IN ('PERCENTAGE', 'FIXED_AMOUNT')),
    discount_value NUMBER(10,2) NOT NULL,
    min_amount NUMBER(19,0) DEFAULT 0,
    max_discount NUMBER(19,0),
    expiry_date TIMESTAMP NOT NULL,
    max_usage NUMBER(10) DEFAULT 0,
    usage_count NUMBER(10) DEFAULT 0,
    max_per_user NUMBER(5) DEFAULT 1,
    eligibility_type VARCHAR2(20) DEFAULT 'ALL_USERS' CHECK (eligibility_type IN ('ALL_USERS', 'RESTRICTED')),
    description VARCHAR2(500),
    created_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    status VARCHAR2(20) DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE', 'INACTIVE', 'EXPIRED')),
    version NUMBER DEFAULT 1 NOT NULL,
    CONSTRAINT check_discount_value_positive CHECK (discount_value > 0),
    CONSTRAINT check_usage_valid CHECK (usage_count <= max_usage)
);

CREATE UNIQUE INDEX idx_discount_code_upper ON discount_codes(UPPER(code));
CREATE INDEX idx_discount_expiry_status ON discount_codes(expiry_date, status);
CREATE INDEX idx_discount_eligibility_type ON discount_codes(eligibility_type);

COMMENT ON TABLE discount_codes IS 'Promotional discount codes with public and user-specific support';
COMMENT ON COLUMN discount_codes.discount_value IS 'Percentage (0-100) or fixed amount in minor units';
COMMENT ON COLUMN discount_codes.eligibility_type IS 'ALL_USERS (public) or RESTRICTED (user-specific)';
COMMENT ON COLUMN discount_codes.description IS 'Human-readable description for admin UI';

-- =====================================================
-- DISCOUNT_CODE_ELIGIBILITY TABLE (NEW)
-- =====================================================
CREATE TABLE discount_code_eligibility (
    id RAW(16) DEFAULT SYS_GUID() PRIMARY KEY,
    discount_code_id RAW(16) NOT NULL,
    user_id RAW(16),
    business_id RAW(16),
    eligibility_type VARCHAR2(20) NOT NULL CHECK (eligibility_type IN ('SPECIFIC_USER', 'SPECIFIC_BUSINESS', 'EMAIL_DOMAIN', 'USER_ROLE')),
    user_email VARCHAR2(255),
    created_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    CONSTRAINT fk_eligibility_discount FOREIGN KEY (discount_code_id) REFERENCES discount_codes(id) ON DELETE CASCADE,
    CONSTRAINT fk_eligibility_user FOREIGN KEY (user_id) REFERENCES users(id),
    CONSTRAINT fk_eligibility_business FOREIGN KEY (business_id) REFERENCES businesses(id)
);

CREATE INDEX idx_eligibility_discount ON discount_code_eligibility(discount_code_id);
CREATE INDEX idx_eligibility_user ON discount_code_eligibility(user_id);
CREATE INDEX idx_eligibility_business ON discount_code_eligibility(business_id);
CREATE INDEX idx_eligibility_type ON discount_code_eligibility(eligibility_type);

COMMENT ON TABLE discount_code_eligibility IS 'Defines who can use specific discount codes (user-specific, business-specific, role-based)';
COMMENT ON COLUMN discount_code_eligibility.eligibility_type IS 'SPECIFIC_USER, SPECIFIC_BUSINESS, EMAIL_DOMAIN, USER_ROLE';
COMMENT ON COLUMN discount_code_eligibility.user_id IS 'Specific user allowed (NULL for other types)';
COMMENT ON COLUMN discount_code_eligibility.business_id IS 'Specific business allowed (NULL for other types)';
COMMENT ON COLUMN discount_code_eligibility.user_email IS 'Email domain pattern (@university.edu) or role name (PREMIUM)';

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
CHECK (final_amount = amount - discount_amount)
CHECK (reserved_amount >= 0)
CHECK (available_credit = credit_limit - outstanding_balance)

-- Enumeration constraints
CHECK (status IN ('ACTIVE', 'SUSPENDED', 'CLOSED'))
CHECK (payment_type IN ('PREPAYMENT', 'POSTPAYMENT', ...))
CHECK (entry_type IN ('DEBIT', 'CREDIT'))

-- Business rules
CHECK (usage_count <= max_usage)
CHECK (paid_installments <= total_installments)
```

### Unique Constraints
```sql
-- Prevent duplicates
UNIQUE (email)  -- users.email
UNIQUE (code)   -- discount_codes.code
UNIQUE (user_id, currency)  -- wallets
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
