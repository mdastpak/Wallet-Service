# GDPR Compliance

## Overview

General Data Protection Regulation (GDPR) compliance implementation for user privacy rights in the Wallet Service.

---

## Data Subject Rights

### 1. Right to Access (Article 15)

**Endpoint**: `GET /v1/users/{user_id}/data-export`

Exports all personal data in machine-readable format (JSON).

**Implementation**:
```java
@GetMapping("/users/{userId}/data-export")
public UserDataExport exportUserData(@PathVariable UUID userId) {
    return UserDataExport.builder()
        .personalInfo(userRepo.findById(userId))
        .wallets(walletRepo.findByUserId(userId))
        .transactions(transactionRepo.findByUserId(userId))
        .auditLogs(auditRepo.findByUserId(userId))
        .build();
}
```

---

### 2. Right to Erasure (Article 17)

**Endpoint**: `DELETE /v1/users/{user_id}/erase`

**Implementation**: Soft delete with anonymization (preserves financial audit trail).

```java
@DeleteMapping("/users/{userId}/erase")
@Transactional
public void eraseUserData(@PathVariable UUID userId) {
    User user = userRepo.findById(userId);

    // Anonymize PII
    user.setEmail("deleted-" + userId + "@anonymous.local");
    user.setFullName("DELETED USER");
    user.setEncryptedPhone(null);
    user.setStatus(UserStatus.DELETED);

    // Preserve transaction history for audit (legal requirement)
    // Mark wallets as closed
    walletRepo.findByUserId(userId).forEach(wallet -> {
        wallet.setStatus(WalletStatus.CLOSED);
    });
}
```

---

### 3. Right to Rectification (Article 16)

**Endpoint**: `PATCH /v1/users/{user_id}`

Users can update their personal information.

---

### 4. Right to Data Portability (Article 20)

Supported via data export endpoint (JSON format).

---

## Consent Management

**Implementation**:
```java
@Entity
public class UserConsent {
    private UUID userId;
    private String consentType; // MARKETING, DATA_PROCESSING, etc.
    private boolean granted;
    private LocalDateTime grantedAt;
    private LocalDateTime revokedAt;
    private String ipAddress;
}
```

---

## Data Retention

| Data Type | Retention Period | Justification |
|-----------|------------------|---------------|
| **Transaction Records** | 7 years | Legal/tax requirement |
| **Audit Logs** | 7 years | Compliance requirement |
| **User PII** | Until deletion request | GDPR Article 17 |
| **Anonymized Analytics** | Indefinite | Not personal data |

---

## Privacy by Design

- Encryption at rest (AES-256)
- PII masking in logs
- Minimal data collection
- Access controls (RBAC)
- Automated data deletion workflows

---

See [Security Architecture](./security-architecture.md) for encryption details.
