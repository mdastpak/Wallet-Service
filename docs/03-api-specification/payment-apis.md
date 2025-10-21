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

## Discount Code APIs

### 1. Validate Discount Code
**POST /v1/discounts/validate**

Validates a discount code for a specific user and transaction amount, including eligibility checks.

**Request**:
```json
{
  "code": "VIP50",
  "amount": 10000,
  "currency": "USD",
  "user_id": "user-123"
}
```

**Response (Success)**:
```json
{
  "valid": true,
  "discount_code_id": "disc-001",
  "code": "VIP50",
  "discount_type": "PERCENTAGE",
  "discount_value": 50.00,
  "discount_amount": 3000,
  "final_amount": 7000,
  "eligibility_type": "RESTRICTED",
  "eligible": true,
  "message": "Discount code applied successfully"
}
```

**Response (Not Eligible)**:
```json
{
  "valid": false,
  "code": "VIP50",
  "eligible": false,
  "error_code": "NOT_ELIGIBLE",
  "message": "User not eligible for this discount code"
}
```

**Response (Other Errors)**:
```json
{
  "valid": false,
  "code": "EXPIRED20",
  "error_code": "EXPIRED",
  "message": "Discount code has expired"
}
```

**Error Codes**:
- `NOT_FOUND`: Discount code does not exist
- `EXPIRED`: Code has passed expiry date
- `INACTIVE`: Code status is not ACTIVE
- `USAGE_LIMIT_REACHED`: Total usage count exceeded
- `USER_LIMIT_REACHED`: User has used code maximum times
- `MIN_AMOUNT_NOT_MET`: Transaction amount below minimum
- `NOT_ELIGIBLE`: User not in eligibility list (for RESTRICTED codes)

---

### 2. Create Discount Code (Admin)
**POST /v1/admin/discounts**

Create a new discount code with optional user-specific eligibility.

**Requires**: `discount:create` scope (Finance Admin role)

**Request (Public Discount)**:
```json
{
  "code": "SUMMER20",
  "discount_type": "PERCENTAGE",
  "discount_value": 20.00,
  "min_amount": 5000,
  "max_discount": 5000,
  "expiry_date": "2025-08-31T23:59:59Z",
  "max_usage": 1000,
  "max_per_user": 1,
  "eligibility_type": "ALL_USERS",
  "description": "Summer sale - 20% off all purchases"
}
```

**Request (User-Specific Discount)**:
```json
{
  "code": "VIP50",
  "discount_type": "PERCENTAGE",
  "discount_value": 50.00,
  "min_amount": 0,
  "max_discount": 10000,
  "expiry_date": "2025-12-31T23:59:59Z",
  "max_usage": 100,
  "max_per_user": 5,
  "eligibility_type": "RESTRICTED",
  "description": "VIP customer exclusive - 50% off",
  "eligibility": [
    {
      "eligibility_type": "SPECIFIC_USER",
      "user_id": "user-john-123"
    },
    {
      "eligibility_type": "SPECIFIC_USER",
      "user_id": "user-jane-456"
    },
    {
      "eligibility_type": "SPECIFIC_USER",
      "user_id": "user-bob-789"
    }
  ]
}
```

**Request (Business-Specific Discount)**:
```json
{
  "code": "ACME25",
  "discount_type": "PERCENTAGE",
  "discount_value": 25.00,
  "min_amount": 0,
  "max_discount": 50000,
  "expiry_date": "2025-12-31T23:59:59Z",
  "max_usage": 500,
  "max_per_user": 10,
  "eligibility_type": "RESTRICTED",
  "description": "Acme Corporation employee discount",
  "eligibility": [
    {
      "eligibility_type": "SPECIFIC_BUSINESS",
      "business_id": "business-acme-corp"
    }
  ]
}
```

**Request (Email Domain Discount)**:
```json
{
  "code": "STUDENT10",
  "discount_type": "PERCENTAGE",
  "discount_value": 10.00,
  "min_amount": 0,
  "max_discount": 2000,
  "expiry_date": "2025-12-31T23:59:59Z",
  "max_usage": 10000,
  "max_per_user": 3,
  "eligibility_type": "RESTRICTED",
  "description": "Student discount for .edu emails",
  "eligibility": [
    {
      "eligibility_type": "EMAIL_DOMAIN",
      "user_email": "@university.edu"
    },
    {
      "eligibility_type": "EMAIL_DOMAIN",
      "user_email": "@college.edu"
    }
  ]
}
```

**Response**:
```json
{
  "id": "disc-001",
  "code": "VIP50",
  "discount_type": "PERCENTAGE",
  "discount_value": 50.00,
  "min_amount": 0,
  "max_discount": 10000,
  "expiry_date": "2025-12-31T23:59:59Z",
  "max_usage": 100,
  "usage_count": 0,
  "max_per_user": 5,
  "eligibility_type": "RESTRICTED",
  "description": "VIP customer exclusive - 50% off",
  "status": "ACTIVE",
  "created_at": "2025-01-15T10:00:00Z",
  "eligibility_count": 3
}
```

---

### 3. Add Eligibility to Discount Code (Admin)
**POST /v1/admin/discounts/{code}/eligibility**

Add additional eligible users/businesses to an existing RESTRICTED discount code.

**Requires**: `discount:manage` scope (Finance Admin role)

**Request**:
```json
{
  "eligibility": [
    {
      "eligibility_type": "SPECIFIC_USER",
      "user_id": "user-new-vip-001"
    }
  ]
}
```

**Response**:
```json
{
  "discount_code_id": "disc-001",
  "code": "VIP50",
  "eligibility_added": 1,
  "total_eligibility_count": 4
}
```

---

### 4. Remove Eligibility from Discount Code (Admin)
**DELETE /v1/admin/discounts/{code}/eligibility/{eligibility_id}**

Remove a specific eligibility rule from a discount code.

**Requires**: `discount:manage` scope (Finance Admin role)

**Response**:
```json
{
  "message": "Eligibility removed successfully",
  "discount_code_id": "disc-001",
  "code": "VIP50",
  "remaining_eligibility_count": 3
}
```

---

### 5. List Eligible Users for Discount Code (Admin)
**GET /v1/admin/discounts/{code}/eligibility**

Retrieve all eligibility rules for a discount code.

**Requires**: `discount:view` scope (Finance Admin role)

**Response**:
```json
{
  "discount_code_id": "disc-001",
  "code": "VIP50",
  "eligibility_type": "RESTRICTED",
  "eligibility": [
    {
      "id": "elig-001",
      "eligibility_type": "SPECIFIC_USER",
      "user_id": "user-john-123",
      "user_email": "john@example.com",
      "user_name": "John Doe",
      "created_at": "2025-01-15T10:00:00Z"
    },
    {
      "id": "elig-002",
      "eligibility_type": "SPECIFIC_USER",
      "user_id": "user-jane-456",
      "user_email": "jane@example.com",
      "user_name": "Jane Smith",
      "created_at": "2025-01-15T10:00:00Z"
    },
    {
      "id": "elig-003",
      "eligibility_type": "SPECIFIC_BUSINESS",
      "business_id": "business-acme-corp",
      "business_name": "Acme Corporation",
      "created_at": "2025-01-16T14:30:00Z"
    }
  ]
}
```

---

### 6. Check User Eligibility (User/Admin)
**GET /v1/discounts/{code}/eligibility/check**

Check if the authenticated user is eligible for a specific discount code.

**Requires**: `discount:check` scope (User or Admin)

**Query Parameters**:
- `user_id` (optional, admin only): Check eligibility for a specific user

**Response (Eligible)**:
```json
{
  "code": "VIP50",
  "eligible": true,
  "eligibility_type": "RESTRICTED",
  "matched_rule": {
    "eligibility_type": "SPECIFIC_USER",
    "user_id": "user-john-123"
  },
  "message": "User is eligible for this discount code"
}
```

**Response (Not Eligible)**:
```json
{
  "code": "VIP50",
  "eligible": false,
  "eligibility_type": "RESTRICTED",
  "message": "User is not eligible for this discount code"
}
```

**Response (Public Code)**:
```json
{
  "code": "SUMMER20",
  "eligible": true,
  "eligibility_type": "ALL_USERS",
  "message": "This is a public discount code available to all users"
}
```

---

## Payment with Discount Code Example

### Create Payment with Discount
**POST /v1/payments**

**Request**:
```json
{
  "payment_type": "PREPAYMENT",
  "source_wallet_id": "wallet-user-123",
  "destination_wallet_id": "wallet-merchant-456",
  "amount": 10000,
  "currency": "USD",
  "discount_code": "VIP50",
  "metadata": {
    "order_id": "order-789",
    "description": "Premium product purchase"
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
  "discount_code": "VIP50",
  "discount_code_id": "disc-001",
  "discount_amount": 3000,
  "final_amount": 7000,
  "created_at": "2025-01-15T10:30:00Z",
  "idempotency_key": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Processing Flow**:
1. Validate discount code "VIP50" exists and is active
2. Check user eligibility (user-123 is in SPECIFIC_USER list)
3. Calculate discount: 50% of $100.00 = $50.00, capped at max_discount $30.00
4. Final amount: $100.00 - $30.00 = $70.00
5. Reserve $70.00 from user wallet (not $100.00)
6. Increment discount usage_count atomically

---

## Error Responses

### Discount Code Not Found
```json
{
  "error": "DISCOUNT_NOT_FOUND",
  "message": "Discount code 'INVALID123' does not exist",
  "status": 404,
  "timestamp": "2025-01-15T10:30:00Z"
}
```

### User Not Eligible
```json
{
  "error": "DISCOUNT_NOT_ELIGIBLE",
  "message": "User is not eligible to use discount code 'VIP50'",
  "status": 403,
  "timestamp": "2025-01-15T10:30:00Z",
  "details": {
    "code": "VIP50",
    "eligibility_type": "RESTRICTED",
    "user_id": "user-999"
  }
}
```

### Usage Limit Reached
```json
{
  "error": "DISCOUNT_USAGE_LIMIT",
  "message": "Discount code 'SUMMER20' has reached its maximum usage limit",
  "status": 429,
  "timestamp": "2025-01-15T10:30:00Z",
  "details": {
    "code": "SUMMER20",
    "max_usage": 1000,
    "usage_count": 1000
  }
}
```

---

## Next Steps

- Review [Payment Types](../02-domain-model/payment-types.md) for discount validation logic
- See [Data Flows](../01-architecture/data-flows.md) for discount application sequence diagrams
- Explore [Database Schema](../02-domain-model/database-schema.md) for discount code tables
