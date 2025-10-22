# Payment APIs

## Overview

Comprehensive payment processing APIs supporting PREPAYMENT, POSTPAYMENT, VALUABLE, and CREDIT payment types with idempotency protection and rollback capabilities.

See complete specifications in:
- [OpenAPI Specification](./openapi-spec.yaml)
- [Data Flows](../01-architecture/data-flows.md)
- [Payment Types](../02-domain-model/payment-types.md)

## Key Endpoints

### 1. Create Payment
**POST /v1/payments**

All payment types share this endpoint with different processing based on `payment_type` field.

### 2. Get Payment Status
**GET /v1/payments/{transaction_id}**

### 3. Rollback Payment
**POST /v1/payments/{transaction_id}/rollback**

Requires `payment:rollback` scope (Finance Admin role).

### 4. Capture Prepayment
**POST /v1/payments/{transaction_id}/capture**

For PREPAYMENT type only.

## Idempotency Requirements

All payment requests MUST include `Idempotency-Key` header.

Refer to [Idempotency Pattern](../06-design-patterns/idempotency-deduplication.md) for complete implementation details.

---

## Payment Type Details

### PREPAYMENT (Reserve and Capture)

**POST /v1/payments**

**Request**:
```json
{
  "payment_type": "PREPAYMENT",
  "source_wallet_id": "wallet-user-123",
  "destination_wallet_id": "wallet-merchant-456",
  "amount": 10000,
  "currency": "USD",
  "metadata": {
    "order_id": "order-789",
    "description": "Product purchase"
  }
}
```

**Headers**:
```
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json
```

**Response**:
```json
{
  "id": "tx-001",
  "payment_type": "PREPAYMENT",
  "status": "PENDING",
  "user_id": "user-123",
  "source_wallet_id": "wallet-user-123",
  "destination_wallet_id": "wallet-merchant-456",
  "amount": 10000,
  "currency": "USD",
  "created_at": "2025-01-15T10:30:00Z",
  "idempotency_key": "550e8400-e29b-41d4-a716-446655440000",
  "expires_at": "2025-01-15T10:35:00Z"
}
```

**Processing Flow**:
1. Validate source wallet has sufficient available balance
2. Create transaction with status PENDING
3. Reserve amount from source wallet (increase reserved_amount)
4. Create RESERVED ledger entries
5. Set TTL timer (5 minutes default)
6. Return transaction ID for capture or cancellation

---

### POSTPAYMENT (Invoice and Settle)

**POST /v1/payments**

**Request**:
```json
{
  "payment_type": "POSTPAYMENT",
  "source_wallet_id": "wallet-user-123",
  "destination_wallet_id": "wallet-supplier-456",
  "amount": 50000,
  "currency": "USD",
  "metadata": {
    "invoice_id": "inv-001",
    "due_date": "2025-02-15",
    "description": "Net-30 payment"
  }
}
```

**Response**:
```json
{
  "id": "tx-002",
  "payment_type": "POSTPAYMENT",
  "status": "PENDING",
  "user_id": "user-123",
  "amount": 50000,
  "currency": "USD",
  "created_at": "2025-01-15T10:30:00Z",
  "due_date": "2025-02-15T00:00:00Z",
  "expires_at": "2025-01-15T11:00:00Z"
}
```

---

### VALUABLE (High-Value Transactions)

**POST /v1/payments**

**Request**:
```json
{
  "payment_type": "VALUABLE",
  "source_wallet_id": "wallet-user-123",
  "destination_wallet_id": "wallet-recipient-456",
  "amount": 1000000,
  "currency": "USD",
  "metadata": {
    "purpose": "Real estate transaction",
    "description": "Property purchase payment"
  }
}
```

**Response**:
```json
{
  "id": "tx-003",
  "payment_type": "VALUABLE",
  "status": "PENDING_APPROVAL",
  "user_id": "user-123",
  "amount": 1000000,
  "currency": "USD",
  "requires_2fa": true,
  "requires_compliance_check": true,
  "created_at": "2025-01-15T10:30:00Z",
  "expires_at": "2025-01-16T10:30:00Z"
}
```

**Processing Flow**:
1. Require 2FA verification
2. Perform compliance and fraud checks
3. Risk scoring and manual review for amounts > $10,000
4. Extended TTL (24 hours)
5. Notification to compliance team

---

### CREDIT (Installment Payments)

**POST /v1/payments**

**Request**:
```json
{
  "payment_type": "CREDIT",
  "destination_wallet_id": "wallet-merchant-456",
  "amount": 30000,
  "currency": "USD",
  "credit_account_id": "credit-user-123",
  "installment_config": {
    "total_installments": 6,
    "frequency": "MONTHLY",
    "first_due_date": "2025-02-15"
  },
  "metadata": {
    "order_id": "order-credit-789"
  }
}
```

**Response**:
```json
{
  "id": "tx-004",
  "payment_type": "CREDIT",
  "status": "PENDING",
  "user_id": "user-123",
  "amount": 30000,
  "currency": "USD",
  "credit_account_id": "credit-user-123",
  "installment_schedule": {
    "total_installments": 6,
    "installment_amount": 5000,
    "frequency": "MONTHLY",
    "first_due_date": "2025-02-15T00:00:00Z",
    "last_due_date": "2025-07-15T00:00:00Z"
  },
  "created_at": "2025-01-15T10:30:00Z"
}
```

---

## Get Payment Status

**GET /v1/payments/{transaction_id}**

**Response**:
```json
{
  "id": "tx-001",
  "payment_type": "PREPAYMENT",
  "status": "CONFIRMED",
  "user_id": "user-123",
  "source_wallet_id": "wallet-user-123",
  "destination_wallet_id": "wallet-merchant-456",
  "amount": 10000,
  "currency": "USD",
  "created_at": "2025-01-15T10:30:00Z",
  "confirmed_at": "2025-01-15T10:31:00Z",
  "ledger_entries": [
    {
      "wallet_id": "wallet-user-123",
      "entry_type": "DEBIT",
      "amount": 10000,
      "status": "CONFIRMED"
    },
    {
      "wallet_id": "wallet-merchant-456",
      "entry_type": "CREDIT",
      "amount": 10000,
      "status": "CONFIRMED"
    }
  ]
}
```

---

## Capture Prepayment

**POST /v1/payments/{transaction_id}/capture**

**Request**:
```json
{
  "amount": 10000
}
```

**Response**:
```json
{
  "id": "tx-001",
  "status": "CONFIRMED",
  "captured_at": "2025-01-15T10:31:00Z",
  "amount": 10000,
  "currency": "USD"
}
```

**Business Rules**:
- Can only capture PENDING prepayments
- Capture amount must be <= reserved amount
- Partial captures allowed
- Auto-capture after TTL expires (configurable)
- Release remaining reserved amount after capture

---

## Rollback Payment

**POST /v1/payments/{transaction_id}/rollback**

**Requires**: `payment:rollback` scope (Finance Admin)

**Request**:
```json
{
  "reason": "Customer requested refund",
  "metadata": {
    "refund_request_id": "ref-001",
    "approved_by": "admin-jane"
  }
}
```

**Response**:
```json
{
  "id": "tx-rollback-001",
  "original_transaction_id": "tx-001",
  "status": "ROLLED_BACK",
  "amount": 10000,
  "currency": "USD",
  "rolled_back_at": "2025-01-15T11:00:00Z",
  "reason": "Customer requested refund"
}
```

**Processing Flow**:
1. Validate original transaction is CONFIRMED
2. Create reverse transaction with ref_transaction_id
3. Create reverse ledger entries (DEBIT → CREDIT, CREDIT → DEBIT)
4. Update original transaction status to ROLLED_BACK
5. Update wallet balances
6. Emit rollback event to Kafka

---

## Error Responses

### Insufficient Balance
```json
{
  "error": "INSUFFICIENT_BALANCE",
  "message": "Wallet does not have sufficient available balance",
  "status": 400,
  "timestamp": "2025-01-15T10:30:00Z",
  "details": {
    "wallet_id": "wallet-user-123",
    "available_balance": 5000,
    "required_amount": 10000,
    "currency": "USD"
  }
}
```

### Invalid Wallet
```json
{
  "error": "INVALID_WALLET",
  "message": "Source or destination wallet is invalid or inactive",
  "status": 400,
  "timestamp": "2025-01-15T10:30:00Z"
}
```

### Idempotency Key Conflict
```json
{
  "error": "IDEMPOTENCY_CONFLICT",
  "message": "Request with this idempotency key already processed",
  "status": 409,
  "timestamp": "2025-01-15T10:30:00Z",
  "details": {
    "existing_transaction_id": "tx-001",
    "idempotency_key": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

### Transaction Not Found
```json
{
  "error": "TRANSACTION_NOT_FOUND",
  "message": "Transaction with ID 'tx-999' does not exist",
  "status": 404,
  "timestamp": "2025-01-15T10:30:00Z"
}
```

---

## Next Steps

- Review [Payment Types](../02-domain-model/payment-types.md) for detailed business logic
- See [Data Flows](../01-architecture/data-flows.md) for sequence diagrams
- Explore [Database Schema](../02-domain-model/database-schema.md) for transaction and ledger tables
- Check [Idempotency Pattern](../06-design-patterns/idempotency-deduplication.md) for duplicate protection
