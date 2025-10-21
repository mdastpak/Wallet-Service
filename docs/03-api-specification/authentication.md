# Authentication & Authorization

## Overview

The Wallet Service supports two authentication mechanisms:
- **OAuth 2.0** for B2C (web/mobile applications)
- **mTLS** for B2B (server-to-server integrations)

---

## OAuth 2.0 (B2C)

### Authorization Code Flow

1. **User Authorization**:
```
GET https://auth.wallet-service.com/oauth/authorize?
    response_type=code&
    client_id=YOUR_CLIENT_ID&
    redirect_uri=YOUR_CALLBACK&
    scope=wallet:read wallet:write&
    state=RANDOM_STATE
```

2. **Token Exchange**:
```bash
curl -X POST https://auth.wallet-service.com/oauth/token \
  -d "grant_type=authorization_code" \
  -d "code=AUTH_CODE" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_SECRET" \
  -d "redirect_uri=YOUR_CALLBACK"
```

3. **Use Access Token**:
```bash
curl -X GET https://api.wallet-service.com/v1/wallets/123/balance \
  -H "Authorization: Bearer ACCESS_TOKEN"
```

---

## mTLS (B2B)

### Certificate Setup

1. Generate client certificate signed by trusted CA
2. Configure certificate in HTTP client
3. Make API requests with mutual TLS

```bash
curl --cert client.pem --key client-key.pem \
     --cacert ca-cert.pem \
     https://api.wallet-service.com/v1/payments
```

---

## Scopes

| Scope | Description |
|-------|-------------|
| `wallet:read` | Read wallet balances |
| `wallet:write` | Create/modify wallets |
| `payment:create` | Create payments |
| `payment:rollback` | Rollback transactions (admin) |
| `balance:read` | Read aggregated balances |

---

Complete implementation details in [Security Architecture](../04-security-compliance/security-architecture.md).
