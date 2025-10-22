# Hierarchical Business Structure - Quick Answer

## ✅ **YES - Your Scenario is VALID!**

### Your Scenario
- **1 Business**: "business-1" with 100 users
- **3 Divisions**: E-commerce, Crypto, Sample-3
- **Requirement**: Each user needs separate wallets per division

### Answer
**YES, this is fully supported** with the **Hierarchical Business Structure** feature!

---

## 🎯 What You Get

### For Each User in business-1

```
User John (john@business-1.com)
│
├── E-commerce Wallets (business_id = line-ecom, child of business-1)
│   ├── USD Wallet (Balance: $10,000)
│   └── EUR Wallet (Balance: €5,000)
│
├── Crypto Wallets (business_id = line-crypto, child of business-1)
│   ├── USD Wallet (Balance: $5,000)
│   ├── BTC Wallet (Balance: 0.5 BTC)
│   └── ETH Wallet (Balance: 2.0 ETH)
│
└── Sample-3 Wallets (business_id = line-sample3, child of business-1)
    └── USD Wallet (Balance: $2,000)

Total: 6 independent wallets with separate balances
```

---

## ✨ Key Features

### 1. Complete Segregation
- ✅ Each division has independent wallets
- ✅ Separate balances per division
- ✅ Independent deposit/withdraw operations
- ✅ No fund mixing between divisions

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
- ✅ **Within same division**: Allowed (e.g., Crypto USD → Crypto BTC)
- ❌ **Between divisions**: Blocked (e.g., E-commerce USD → Crypto USD)
- ✅ **With approval**: Admin can approve cross-division transfers

---

## 📋 Database Structure

### Hierarchical Business Table

**Parent Business:**
```sql
INSERT INTO businesses (id, parent_id, name, business_type, country, email)
VALUES ('biz-001', NULL, 'Acme Corp', 'CORPORATION', 'US', 'contact@acme.com');
```

**Child Divisions:**
```sql
INSERT INTO businesses (id, parent_id, name, code) VALUES
  ('line-ecom', 'biz-001', 'E-commerce', 'ECOMMERCE'),
  ('line-crypto', 'biz-001', 'Crypto', 'CRYPTO'),
  ('line-sample3', 'biz-001', 'Sample-3', 'SAMPLE3');
```

**Wallets Reference Either Parent or Child:**
```sql
-- E-commerce division wallet
INSERT INTO wallets (user_id, business_id, currency, wallet_type)
VALUES ('user-john', 'line-ecom', 'USD', 'BUSINESS');

-- Crypto division wallet
INSERT INTO wallets (user_id, business_id, currency, wallet_type)
VALUES ('user-john', 'line-crypto', 'USD', 'BUSINESS');
```

---

## 🔍 Key Relationships

```
BUSINESS (parent_id = NULL) ──┐
  │                             │ parent-child
  └─> BUSINESS (parent_id = biz-001) ─── line-ecom
  └─> BUSINESS (parent_id = biz-001) ─── line-crypto
  └─> BUSINESS (parent_id = biz-001) ─── line-sample3

WALLET (business_id = line-ecom) ──> references child business
WALLET (business_id = line-crypto) ──> references child business
WALLET (business_id = line-sample3) ──> references child business
```

---

## 🆚 What Changed from Previous Version

### Before (Version 2.0)
- Separate `BUSINESS` and `BUSINESS_LINE` tables
- `WALLET.business_line_id` column
- Complex discount code eligibility system

### After (Version 3.0)
- **Single `BUSINESS` table** with self-referencing `parent_id`
- `WALLET.business_id` references either parent or child business
- **Removed discount code complexity**
- **Simplified schema** with same functionality

---

## ✅ Benefits

1. **Simpler Schema**: One table instead of two
2. **Flexible Hierarchy**: Can support multi-level hierarchies in future
3. **Same Functionality**: All previous features maintained
4. **Better Performance**: Fewer JOINs needed
5. **Cleaner Data Model**: Standard parent-child pattern

---

## 📚 Related Documentation

- **[Complete ERD](./COMPLETE_ERD.md)** - Full entity relationship diagram
- **[Database Schema](./docs/02-domain-model/database-schema.md)** - Complete SQL DDL
- **[Wallet APIs](./docs/03-api-specification/wallet-apis.md)** - API endpoints
- **[Payment APIs](./docs/03-api-specification/payment-apis.md)** - Payment processing

---

**Version**: 3.0 (Hierarchical Business Structure)
**Last Updated**: October 2025
**Status**: Production-Ready Design
