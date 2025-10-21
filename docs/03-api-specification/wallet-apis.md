# Wallet APIs

## Overview

RESTful API endpoints for wallet management operations including creation, balance queries, and wallet lifecycle management.

**Base URL**: `https://api.wallet-service.com/v1`

**Authentication**: OAuth 2.0 Bearer Token or mTLS (B2B)

---

## Endpoints

### 1. Create Wallet

**Endpoint**: `POST /v1/wallets`

**Description**: Create a new wallet for a user in a specific currency.

**Authorization**: `wallet:write` scope

**Request**:
```json
{
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "currency": "USD",
  "wallet_type": "STANDARD"
}
```

**Response** (201 Created):
```json
{
  "wallet_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "business_id": null,
  "currency": "USD",
  "wallet_type": "STANDARD",
  "balance": "0.00",
  "available_balance": "0.00",
  "reserved_amount": "0.00",
  "status": "ACTIVE",
  "created_at": "2025-01-15T10:30:00Z"
}
```

**Error Responses**:
- `409 Conflict` - Wallet already exists for this user/currency
- `400 Bad Request` - Invalid currency or wallet type
- `401 Unauthorized` - Missing or invalid token

**cURL Example**:
```bash
curl -X POST https://api.wallet-service.com/v1/wallets \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "currency": "USD",
    "wallet_type": "STANDARD"
  }'
```

---

### 2. Get Wallet Balance

**Endpoint**: `GET /v1/wallets/{wallet_id}/balance`

**Description**: Retrieve current balance for a specific wallet (cached).

**Authorization**: `wallet:read` scope

**Path Parameters**:
- `wallet_id` (UUID, required): Wallet identifier

**Query Parameters**:
- `refresh` (boolean, optional): Force balance recomputation (default: false)

**Response** (200 OK):
```json
{
  "wallet_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "currency": "USD",
  "balance": "1500.00",
  "available_balance": "1450.00",
  "reserved_amount": "50.00",
  "last_updated": "2025-01-15T10:35:22Z",
  "cached": true
}
```

**cURL Example**:
```bash
curl -X GET "https://api.wallet-service.com/v1/wallets/${WALLET_ID}/balance?refresh=false" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

---

### 3. List User Wallets

**Endpoint**: `GET /v1/users/{user_id}/wallets`

**Description**: List all wallets for a specific user.

**Authorization**: `wallet:read` scope

**Path Parameters**:
- `user_id` (UUID, required): User identifier

**Query Parameters**:
- `status` (string, optional): Filter by status (ACTIVE, FROZEN, CLOSED)
- `currency` (string, optional): Filter by currency

**Response** (200 OK):
```json
{
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "wallets": [
    {
      "wallet_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "currency": "USD",
      "wallet_type": "STANDARD",
      "balance": "1500.00",
      "status": "ACTIVE"
    },
    {
      "wallet_id": "9f4b2a1c-8e3d-4b2a-9c1e-7f8d6e5c4b3a",
      "currency": "EUR",
      "wallet_type": "SAVINGS",
      "balance": "850.50",
      "status": "ACTIVE"
    }
  ],
  "total_count": 2
}
```

---

### 4. Freeze Wallet

**Endpoint**: `POST /v1/wallets/{wallet_id}/freeze`

**Description**: Temporarily freeze a wallet (no transactions allowed).

**Authorization**: `wallet:admin` scope

**Request**:
```json
{
  "reason": "Suspicious activity detected",
  "duration_hours": 24
}
```

**Response** (200 OK):
```json
{
  "wallet_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "status": "FROZEN",
  "frozen_at": "2025-01-15T11:00:00Z",
  "unfreeze_at": "2025-01-16T11:00:00Z",
  "reason": "Suspicious activity detected"
}
```

---

### 5. Close Wallet

**Endpoint**: `DELETE /v1/wallets/{wallet_id}`

**Description**: Permanently close a wallet (balance must be zero).

**Authorization**: `wallet:admin` scope

**Response** (200 OK):
```json
{
  "wallet_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "status": "CLOSED",
  "closed_at": "2025-01-15T12:00:00Z",
  "final_balance": "0.00"
}
```

**Error Responses**:
- `400 Bad Request` - Wallet has non-zero balance
- `409 Conflict` - Wallet has pending transactions

---

## Complete OpenAPI 3.0 Specification (Excerpt)

```yaml
openapi: 3.0.3
info:
  title: Wallet Service API
  version: 1.0.0
  description: Enterprise wallet management system
servers:
  - url: https://api.wallet-service.com/v1
    description: Production
  - url: https://staging.wallet-service.com/v1
    description: Staging
security:
  - oauth2: []
  - mtls: []

paths:
  /wallets:
    post:
      summary: Create Wallet
      operationId: createWallet
      tags:
        - Wallets
      security:
        - oauth2: [wallet:write]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateWalletRequest'
      responses:
        '201':
          description: Wallet created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/WalletResponse'
        '409':
          $ref: '#/components/responses/Conflict'

  /wallets/{wallet_id}/balance:
    get:
      summary: Get Wallet Balance
      operationId: getWalletBalance
      tags:
        - Wallets
      security:
        - oauth2: [wallet:read]
      parameters:
        - name: wallet_id
          in: path
          required: true
          schema:
            type: string
            format: uuid
        - name: refresh
          in: query
          schema:
            type: boolean
            default: false
      responses:
        '200':
          description: Balance retrieved successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/BalanceResponse'

components:
  schemas:
    CreateWalletRequest:
      type: object
      required:
        - user_id
        - currency
      properties:
        user_id:
          type: string
          format: uuid
        currency:
          type: string
          pattern: '^[A-Z]{3}$'
          example: USD
        wallet_type:
          type: string
          enum: [STANDARD, SAVINGS, BUSINESS, ESCROW]
          default: STANDARD

    WalletResponse:
      type: object
      properties:
        wallet_id:
          type: string
          format: uuid
        user_id:
          type: string
          format: uuid
        business_id:
          type: string
          format: uuid
          nullable: true
        currency:
          type: string
        wallet_type:
          type: string
        balance:
          type: string
          pattern: '^\d+\.\d{2}$'
        available_balance:
          type: string
        reserved_amount:
          type: string
        status:
          type: string
          enum: [ACTIVE, FROZEN, CLOSED]
        created_at:
          type: string
          format: date-time

    BalanceResponse:
      type: object
      properties:
        wallet_id:
          type: string
          format: uuid
        currency:
          type: string
        balance:
          type: string
        available_balance:
          type: string
        reserved_amount:
          type: string
        last_updated:
          type: string
          format: date-time
        cached:
          type: boolean

  securitySchemes:
    oauth2:
      type: oauth2
      flows:
        authorizationCode:
          authorizationUrl: https://auth.wallet-service.com/oauth/authorize
          tokenUrl: https://auth.wallet-service.com/oauth/token
          scopes:
            wallet:read: Read wallet information
            wallet:write: Create and modify wallets
            wallet:admin: Administrative operations
    mtls:
      type: mutualTLS
      description: Mutual TLS for B2B integrations

  responses:
    Conflict:
      description: Resource conflict
      content:
        application/json:
          schema:
            type: object
            properties:
              error:
                type: string
                example: "Wallet already exists"
              code:
                type: string
                example: "WALLET_DUPLICATE"
```

---

## Rate Limiting

| Endpoint | Rate Limit | Window |
|----------|-----------|--------|
| `POST /wallets` | 10 req/min | Per user |
| `GET /wallets/{id}/balance` | 100 req/min | Per user |
| `GET /users/{id}/wallets` | 50 req/min | Per user |
| Admin endpoints | 20 req/min | Per token |

**Rate Limit Headers**:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 87
X-RateLimit-Reset: 1610728800
```

---

## Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `WALLET_NOT_FOUND` | 404 | Wallet does not exist |
| `WALLET_DUPLICATE` | 409 | Wallet already exists for user/currency |
| `WALLET_FROZEN` | 403 | Wallet is frozen |
| `INVALID_CURRENCY` | 400 | Unsupported currency code |
| `INSUFFICIENT_BALANCE` | 402 | Wallet balance too low |

---

For complete payment and aggregation APIs, see:
- [Payment APIs](./payment-apis.md)
- [Aggregation APIs](./aggregation-apis.md)
- [Authentication](./authentication.md)
