# Hierarchical Business Structure - Quick Answer

## ✅ **YES - Your Scenario is VALID!**

### Your Scenario
- **1 Business**: "business-1" with 100 users
- **3 Divisions**: E-commerce, Crypto, Operations
- **Requirement**: Each user needs separate wallets per division

### Answer
**YES, this is fully supported** with the **Hierarchical Business Structure** feature!

---

## 🎯 What You Get

### For Each User in business-1

```
User John (john@business-1.com)
│
├── E-commerce Wallets (business_id = 3d2f8a9e-12ab-4c8d-9f6e-7a8b9c0d1e2f, child of business-1)
│   ├── USD Wallet (Balance: $10,000)
│   └── EUR Wallet (Balance: €5,000)
│
├── Crypto Wallets (business_id = 8b4e1c2a-45de-4f7a-89ab-0c1d2e3f4a5b, child of business-1)
│   ├── USD Wallet (Balance: $5,000)
│   ├── BTC Wallet (Balance: 0.5 BTC)
│   └── ETH Wallet (Balance: 2.0 ETH)
│
└── Operations Wallets (business_id = 6f3d9b1c-89cd-4e5f-a1b2-c3d4e5f6a7b8, child of business-1)
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
→ Crypto and Operations wallets unchanged
```

**Withdraw Example:**
```
John withdraws $500 from Crypto USD wallet
→ Only Crypto USD balance decreases
→ E-commerce and Operations wallets unchanged
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
VALUES ('f47ac10b-58cc-4372-a567-0e02b2c3d479', NULL, 'Acme Corp', 'CORPORATION', 'US', 'contact@acme.com');
```

**Child Divisions:**
```sql
INSERT INTO businesses (id, parent_id, name, code) VALUES
  ('3d2f8a9e-12ab-4c8d-9f6e-7a8b9c0d1e2f', 'f47ac10b-58cc-4372-a567-0e02b2c3d479', 'E-commerce', 'ECOMMERCE'),
  ('8b4e1c2a-45de-4f7a-89ab-0c1d2e3f4a5b', 'f47ac10b-58cc-4372-a567-0e02b2c3d479', 'Crypto', 'CRYPTO'),
  ('6f3d9b1c-89cd-4e5f-a1b2-c3d4e5f6a7b8', 'f47ac10b-58cc-4372-a567-0e02b2c3d479', 'Operations', 'OPERATIONS');
```

**Wallets Reference Either Parent or Child:**
```sql
-- E-commerce division wallet
INSERT INTO wallets (user_id, business_id, currency, wallet_type)
VALUES ('e5a9c2f1-3b4d-4c8e-9a1f-2b3c4d5e6f7a', '3d2f8a9e-12ab-4c8d-9f6e-7a8b9c0d1e2f', 'USD', 'BUSINESS');

-- Crypto division wallet
INSERT INTO wallets (user_id, business_id, currency, wallet_type)
VALUES ('e5a9c2f1-3b4d-4c8e-9a1f-2b3c4d5e6f7a', '8b4e1c2a-45de-4f7a-89ab-0c1d2e3f4a5b', 'USD', 'BUSINESS');
```

---

## 🔍 Key Relationships

```
BUSINESS (parent_id = NULL) ──┐
  │                             │ parent-child
  └─> BUSINESS (parent_id = f47ac10b-...) ─── E-commerce (3d2f8a9e-...)
  └─> BUSINESS (parent_id = f47ac10b-...) ─── Crypto (8b4e1c2a-...)
  └─> BUSINESS (parent_id = f47ac10b-...) ─── Operations (6f3d9b1c-...)

WALLET (business_id = 3d2f8a9e-...) ──> references E-commerce child
WALLET (business_id = 8b4e1c2a-...) ──> references Crypto child
WALLET (business_id = 6f3d9b1c-...) ──> references Operations child
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
