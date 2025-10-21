# Test Data Management

## Overview

Strategies for managing test data, fixtures, and factories for comprehensive testing.

---

## Test Data Builders

```java
public class TestDataBuilder {

    public static class UserBuilder {
        private String email = "test@example.com";
        private String fullName = "Test User";
        private KycStatus kycStatus = KycStatus.APPROVED;

        public UserBuilder withEmail(String email) {
            this.email = email;
            return this;
        }

        public UserBuilder withKycStatus(KycStatus status) {
            this.kycStatus = status;
            return this;
        }

        public User build() {
            User user = new User();
            user.setId(UUID.randomUUID());
            user.setEmail(email);
            user.setFullName(fullName);
            user.setKycStatus(kycStatus);
            user.setStatus(UserStatus.ACTIVE);
            user.setCreatedAt(LocalDateTime.now());
            return user;
        }
    }

    public static UserBuilder aUser() {
        return new UserBuilder();
    }
}
```

**Usage**:
```java
User user = aUser()
    .withEmail("premium@example.com")
    .withKycStatus(KycStatus.APPROVED)
    .build();
```

---

## Database Seeding

```java
@Component
@Profile("test")
public class TestDataSeeder implements ApplicationListener<ContextRefreshedEvent> {

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        seedTestData();
    }

    @Transactional
    public void seedTestData() {
        // Create test users
        User user1 = aUser().withEmail("user1@test.com").build();
        User user2 = aUser().withEmail("user2@test.com").build();
        userRepo.saveAll(List.of(user1, user2));

        // Create wallets
        Wallet wallet1 = aWallet().forUser(user1).withCurrency("USD").build();
        Wallet wallet2 = aWallet().forUser(user2).withCurrency("EUR").build();
        walletRepo.saveAll(List.of(wallet1, wallet2));

        // Fund wallets
        fundWallet(wallet1, new BigDecimal("1000.00"));
        fundWallet(wallet2, new BigDecimal("500.00"));
    }
}
```

---

## Realistic Test Scenarios

### Scenario: New User Onboarding
```java
@Test
void newUserJourneyTest() {
    // 1. User signs up
    User user = signUp("newuser@example.com");

    // 2. Complete KYC
    completeKyc(user);

    // 3. Create wallets
    Wallet usdWallet = createWallet(user, "USD");
    Wallet eurWallet = createWallet(user, "EUR");

    // 4. Fund account (external deposit)
    deposit(usdWallet, new BigDecimal("1000.00"));

    // 5. Make first payment
    PaymentResponse payment = makePayment(usdWallet, new BigDecimal("50.00"));

    // Assertions
    assertThat(payment.getStatus()).isEqualTo(TransactionStatus.CONFIRMED);
    assertThat(getBalance(usdWallet)).isEqualByComparingTo("950.00");
}
```

---

## Test Data Cleanup

```java
@AfterEach
void cleanup() {
    transactionRepo.deleteAll();
    ledgerEntryRepo.deleteAll();
    walletRepo.deleteAll();
    userRepo.deleteAll();

    // Clear Redis cache
    redisTemplate.getConnectionFactory().getConnection().flushAll();
}
```

---

## Sample Datasets

### Currencies
```java
List<String> CURRENCIES = List.of("USD", "EUR", "GBP", "JPY", "BTC", "ETH");
```

### Payment Amounts
```java
List<BigDecimal> AMOUNTS = List.of(
    new BigDecimal("10.00"),    // Small payment
    new BigDecimal("100.00"),   // Medium payment
    new BigDecimal("1000.00"),  // Large payment
    new BigDecimal("15000.00")  // Valuable payment (triggers extra checks)
);
```

---

See [Test Strategy](./test-strategy.md) for complete testing approach.
