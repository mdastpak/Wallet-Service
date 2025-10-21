# Business Line Wallet Support - Quick Answer

## ✅ **YES - Your Scenario is VALID!**

### Your Scenario
- **1 Business**: "business-1" with 100 users
- **3 Business Lines**: E-commerce, Crypto, Sample-3
- **Requirement**: Each user needs separate wallets per business line

### Answer
**YES, this is fully supported** with the **Business Line Wallet Segregation** feature!

---

## 🎯 What You Get

### For Each User in business-1

```
User John (john@business-1.com)
│
├── E-commerce Wallets (business_line_id = line-ecom)
│   ├── USD Wallet (Balance: $10,000)
│   └── EUR Wallet (Balance: €5,000)
│
├── Crypto Wallets (business_line_id = line-crypto)
│   ├── USD Wallet (Balance: $5,000)
│   ├── BTC Wallet (Balance: 0.5 BTC)
│   └── ETH Wallet (Balance: 2.0 ETH)
│
└── Sample-3 Wallets (business_line_id = line-sample3)
    └── USD Wallet (Balance: $2,000)

Total: 6 independent wallets with separate balances
```

---

## ✨ Key Features

### 1. Complete Segregation
- ✅ Each business line has independent wallets
- ✅ Separate balances per business line
- ✅ Independent deposit/withdraw operations
- ✅ No fund mixing between business lines

### 2. Operations Supported

**Deposit Example:**
```
John deposits $1,000 to E-commerce USD wallet
→ Only E-commerce USD balance increases
→ Crypto and Sample-3 wallets unchanged
```

**Withdraw Example:**
```
John withdraws $500 from Crypto USD wallet
→ Only Crypto USD balance decreases
→ E-commerce and Sample-3 wallets unchanged
```

**Transfer Rules:**
- ✅ **Within same business line**: Allowed (e.g., Crypto USD → Crypto BTC)
- ❌ **Between business lines**: Blocked (e.g., E-commerce USD → Crypto USD)
- ✅ **With approval**: Admin can approve cross-business-line transfers

### 3. Business Line-Specific Discount Codes

```
Discount "ECOM20" (20% off for E-commerce division)
  eligibility_type: SPECIFIC_BUSINESS_LINE
  business_line_id: line-ecom

When John uses "ECOM20":
  ✅ Valid for E-commerce wallets
  ❌ Invalid for Crypto wallets
  ❌ Invalid for Sample-3 wallets
```

---

## 📊 Database Design

### New Table: business_lines

```sql
CREATE TABLE business_lines (
    id RAW(16) DEFAULT SYS_GUID() PRIMARY KEY,
    business_id RAW(16) NOT NULL,  -- Parent business (business-1)
    name VARCHAR2(100) NOT NULL,   -- "E-commerce", "Crypto", "Sample-3"
    code VARCHAR2(50) NOT NULL,    -- "ECOMMERCE", "CRYPTO", "SAMPLE3"
    description VARCHAR2(500),
    is_active NUMBER(1) DEFAULT 1,
    created_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    updated_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    CONSTRAINT fk_business_line_business FOREIGN KEY (business_id) REFERENCES businesses(id),
    CONSTRAINT uk_business_line_code UNIQUE (business_id, code)
);

-- Setup for business-1
INSERT INTO business_lines (id, business_id, name, code) VALUES
  ('line-ecom', 'biz-001', 'E-commerce', 'ECOMMERCE'),
  ('line-crypto', 'biz-001', 'Crypto', 'CRYPTO'),
  ('line-sample3', 'biz-001', 'Sample-3', 'SAMPLE3');
```

### Enhanced Table: wallets

```sql
CREATE TABLE wallets (
    id RAW(16) DEFAULT SYS_GUID() PRIMARY KEY,
    user_id RAW(16) NOT NULL,
    business_id RAW(16),           -- business-1
    business_line_id RAW(16),      -- NEW: E-commerce, Crypto, or Sample-3
    currency CHAR(3) NOT NULL,
    wallet_type VARCHAR2(20) DEFAULT 'STANDARD',
    reserved_amount NUMBER(19,0) DEFAULT 0 NOT NULL,
    wallet_name VARCHAR2(255),
    is_active NUMBER(1) DEFAULT 1,
    created_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    updated_at TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL,
    status VARCHAR2(20) DEFAULT 'ACTIVE',
    version NUMBER DEFAULT 1 NOT NULL,
    CONSTRAINT fk_wallet_user FOREIGN KEY (user_id) REFERENCES users(id),
    CONSTRAINT fk_wallet_business FOREIGN KEY (business_id) REFERENCES businesses(id),
    CONSTRAINT fk_wallet_business_line FOREIGN KEY (business_line_id) REFERENCES business_lines(id),
    CONSTRAINT check_business_line_requires_business CHECK (
        business_line_id IS NULL OR business_id IS NOT NULL
    )
);

-- UNIQUE CONSTRAINT: Each user can have ONE wallet per (business_id, business_line_id, currency, wallet_type)
CREATE UNIQUE INDEX idx_wallet_user_business_line_currency ON wallets(
    user_id,
    COALESCE(business_id, RAW '00000000000000000000000000000000'),
    COALESCE(business_line_id, RAW '00000000000000000000000000000000'),
    currency,
    wallet_type
);
```

---

## 📋 API Examples

### List Wallets Grouped by Business Line

```http
GET /v1/users/{user_id}/wallets?group_by=business_line
Authorization: Bearer {token}

Response: 200 OK
{
  "user_id": "user-john",
  "business_id": "biz-001",
  "business_name": "business-1",
  "total_wallets": 6,
  "wallet_groups": [
    {
      "business_line_id": "line-ecom",
      "business_line_name": "E-commerce",
      "wallets": [
        {
          "id": "wallet-001",
          "currency": "USD",
          "wallet_name": "E-commerce USD",
          "balance": 1000000,
          "available_balance": 1000000
        },
        {
          "id": "wallet-002",
          "currency": "EUR",
          "wallet_name": "E-commerce EUR",
          "balance": 500000,
          "available_balance": 500000
        }
      ]
    },
    {
      "business_line_id": "line-crypto",
      "business_line_name": "Crypto",
      "wallets": [
        {
          "id": "wallet-003",
          "currency": "USD",
          "wallet_name": "Crypto USD",
          "balance": 500000,
          "available_balance": 500000
        },
        {
          "id": "wallet-004",
          "currency": "BTC",
          "wallet_name": "Crypto BTC",
          "balance": 50000000,
          "available_balance": 50000000
        }
      ]
    },
    {
      "business_line_id": "line-sample3",
      "business_line_name": "Sample-3",
      "wallets": [
        {
          "id": "wallet-005",
          "currency": "USD",
          "wallet_name": "Sample-3 USD",
          "balance": 200000,
          "available_balance": 200000
        }
      ]
    }
  ]
}
```

### Create Business Line Wallet

```http
POST /v1/wallets
Content-Type: application/json
Authorization: Bearer {token}

{
  "user_id": "user-john",
  "business_id": "biz-001",
  "business_line_id": "line-ecom",  // E-commerce line
  "currency": "GBP",
  "wallet_type": "BUSINESS",
  "wallet_name": "E-commerce GBP"
}

Response: 201 Created
{
  "id": "wallet-006",
  "user_id": "user-john",
  "business_id": "biz-001",
  "business_name": "business-1",
  "business_line_id": "line-ecom",
  "business_line_name": "E-commerce",
  "currency": "GBP",
  "wallet_name": "E-commerce GBP",
  "balance": 0,
  "status": "ACTIVE"
}
```

### Deposit to Business Line Wallet

```http
POST /v1/transactions/deposit
Content-Type: application/json
Authorization: Bearer {token}

{
  "destination_wallet_id": "wallet-003",  // Crypto USD wallet
  "amount": 100000,  // $1,000.00
  "currency": "USD",
  "description": "Crypto division payment"
}

Response: 200 OK
{
  "transaction_id": "tx-001",
  "amount": 100000,
  "currency": "USD",
  "status": "CONFIRMED",
  "wallet": {
    "id": "wallet-003",
    "business_line_name": "Crypto",
    "new_balance": 600000  // $6,000.00 (was $5,000 + $1,000)
  }
}

// E-commerce and Sample-3 wallets unchanged
```

---

## 🎯 Your business-1 Structure

```
business-1 (100 users)
│
├── E-commerce Line
│   ├── User John: E-commerce USD, E-commerce EUR
│   ├── User Alice: E-commerce USD
│   ├── User Bob: E-commerce USD, E-commerce GBP
│   └── ... 97 more users
│
├── Crypto Line
│   ├── User John: Crypto USD, Crypto BTC, Crypto ETH
│   ├── User Alice: Crypto USD, Crypto BTC
│   ├── User Bob: Crypto USD
│   └── ... 97 more users
│
└── Sample-3 Line
    ├── User John: Sample-3 USD
    ├── User Alice: Sample-3 USD, Sample-3 EUR
    ├── User Bob: Sample-3 USD
    └── ... 97 more users

Total Potential: 100 users × 3 lines × N currencies = Highly Scalable
```

---

## ✅ Benefits for business-1

1. **Complete Segregation**: E-commerce, Crypto, and Sample-3 operate independently
2. **Financial Clarity**: Each division has clear balance reporting
3. **Operational Independence**: Each business line can manage its own finances
4. **Compliance**: Separate audit trails per business line
5. **Scalability**: Add new business lines without affecting existing wallets
6. **User Flexibility**: Users can work across multiple business lines
7. **Discount Management**: Business line-specific promotional campaigns

---

## 🚀 Implementation Status

### Completed
- ✅ Database schema design (business_lines table + enhanced wallets table)
- ✅ Unique constraint for business line segregation
- ✅ Documentation with examples
- ✅ API endpoint designs
- ✅ Business rules defined

### Next Steps
1. Implement database migrations
2. Update wallet service APIs
3. Implement business line-specific discount validation
4. Add cross-business-line transfer approval workflow
5. Create test data for business-1 scenario
6. Update UI to show business line grouping

---

## 📖 Complete Documentation

For full details, see:
- **[BUSINESS_LINE_WALLET_SUPPORT.md](./BUSINESS_LINE_WALLET_SUPPORT.md)** - Complete feature guide with code examples
- **[database-schema.md](./docs/02-domain-model/database-schema.md)** - Updated DDL with business_lines table
- **[multi-business-wallets.md](./docs/02-domain-model/multi-business-wallets.md)** - Original multi-business wallet guide

---

## 💡 Quick Example

**Before (Not Supported):**
```
❌ John has 1 USD wallet for all of business-1
   - E-commerce, Crypto, Sample-3 all share the same balance
   - No segregation between business lines
```

**After (With Business Line Support):**
```
✅ John has 3 separate USD wallets:
   - E-commerce USD: $10,000 (independent)
   - Crypto USD: $5,000 (independent)
   - Sample-3 USD: $2,000 (independent)
   - Total: $17,000 across 3 business lines
```

---

**Status**: ✅ **READY FOR IMPLEMENTATION**

**Your Question**: Can each user have multiple wallets per business with separate operations for different business lines (e-commerce, crypto, sample-3)?

**Answer**: **YES! Fully supported with the Business Line Wallet Segregation feature.**

---

**Next Step**: Review the complete documentation in [BUSINESS_LINE_WALLET_SUPPORT.md](./BUSINESS_LINE_WALLET_SUPPORT.md) and confirm this matches your requirements for business-1.
