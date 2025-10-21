# Test Strategy

## Overview

Comprehensive testing strategy covering unit, integration, contract, load, and chaos testing for the Wallet Service.

---

## Testing Pyramid

```
       /\
      /  \  E2E Tests (5%)
     /____\
    /      \  Integration Tests (25%)
   /________\
  /          \  Unit Tests (70%)
 /____________\
```

---

## Test Categories

### 1. Unit Tests (70%)

**Scope**: Individual classes and methods in isolation

**Framework**: JUnit 5 + Mockito

**Example**:
```java
@ExtendWith(MockitoExtension.class)
class PaymentServiceTest {

    @Mock
    private TransactionRepository transactionRepo;

    @Mock
    private PaymentGatewayClient gatewayClient;

    @InjectMocks
    private PaymentService paymentService;

    @Test
    void shouldCreatePaymentSuccessfully() {
        // Given
        PaymentRequest request = createPaymentRequest(100.00);
        when(transactionRepo.save(any())).thenAnswer(invocation -> invocation.getArgument(0));
        when(gatewayClient.processPayment(any())).thenReturn(successResponse());

        // When
        PaymentResponse response = paymentService.processPayment(request);

        // Then
        assertThat(response.getStatus()).isEqualTo(TransactionStatus.CONFIRMED);
        verify(transactionRepo).save(any(Transaction.class));
        verify(gatewayClient).processPayment(any());
    }

    @Test
    void shouldHandleInsufficientFunds() {
        // Given
        PaymentRequest request = createPaymentRequest(1000000.00);

        // When/Then
        assertThatThrownBy(() -> paymentService.processPayment(request))
            .isInstanceOf(InsufficientFundsException.class)
            .hasMessageContaining("Insufficient balance");
    }
}
```

---

### 2. Integration Tests (25%)

**Scope**: Multiple components working together with real dependencies

**Framework**: Spring Boot Test + Testcontainers

**Example**:
```java
@SpringBootTest
@Testcontainers
class PaymentIntegrationTest {

    @Container
    static OracleContainer oracle = new OracleContainer("gvenzl/oracle-xe:21-slim");

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);

    @Autowired
    private PaymentService paymentService;

    @Autowired
    private TransactionRepository transactionRepo;

    @Test
    @Transactional
    void shouldProcessPaymentEndToEnd() {
        // Given
        User user = createAndSaveUser();
        Wallet wallet = createAndSaveWallet(user);
        fundWallet(wallet, new BigDecimal("1000.00"));

        PaymentRequest request = PaymentRequest.builder()
            .userId(user.getId())
            .sourceWalletId(wallet.getId())
            .amount(new BigDecimal("100.00"))
            .currency("USD")
            .paymentType(PaymentType.PREPAYMENT)
            .build();

        // When
        PaymentResponse response = paymentService.processPayment(
            request, UUID.randomUUID().toString(), user.getId(), "/v1/payments"
        );

        // Then
        assertThat(response.getStatus()).isEqualTo(TransactionStatus.CONFIRMED);

        // Verify database state
        Transaction tx = transactionRepo.findById(response.getTransactionId()).orElseThrow();
        assertThat(tx.getStatus()).isEqualTo(TransactionStatus.CONFIRMED);
        assertThat(tx.getAmount()).isEqualTo(10000L);  // $100.00 in cents

        // Verify ledger entries
        List<LedgerEntry> entries = ledgerRepo.findByTransactionId(tx.getId());
        assertThat(entries).hasSize(2);
        assertThat(entries).extracting(LedgerEntry::getEntryType)
            .containsExactlyInAnyOrder(EntryType.DEBIT, EntryType.CREDIT);
    }
}
```

---

### 3. Contract Tests

**Scope**: API contracts between services

**Framework**: Spring Cloud Contract / Pact

**Example** (Consumer):
```java
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "payment-gateway")
class PaymentGatewayContractTest {

    @Pact(consumer = "wallet-service")
    public RequestResponsePact processPaymentPact(PactDslWithProvider builder) {
        return builder
            .given("gateway is available")
            .uponReceiving("payment request")
                .path("/process")
                .method("POST")
                .body(new PactDslJsonBody()
                    .stringValue("amount", "100.00")
                    .stringValue("currency", "USD"))
            .willRespondWith()
                .status(200)
                .body(new PactDslJsonBody()
                    .stringValue("transactionId", "gateway-tx-123")
                    .stringValue("status", "SUCCESS"))
            .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "processPaymentPact")
    void testProcessPayment(MockServer mockServer) {
        PaymentGatewayClient client = new PaymentGatewayClient(mockServer.getUrl());
        GatewayResponse response = client.processPayment(createRequest());

        assertThat(response.getStatus()).isEqualTo("SUCCESS");
    }
}
```

---

### 4. Load Tests

**Scope**: Performance under load

**Framework**: Gatling / JMeter

**Example** (Gatling):
```scala
class PaymentLoadTest extends Simulation {

  val httpProtocol = http
    .baseUrl("https://api.wallet-service.com")
    .header("Authorization", "Bearer ${ACCESS_TOKEN}")

  val scn = scenario("Payment Processing")
    .exec(http("Create Payment")
      .post("/v1/payments")
      .header("Idempotency-Key", "${UUID}")
      .body(StringBody("""{"amount": 100.00, "currency": "USD"}"""))
      .check(status.is(200)))

  setUp(
    scn.inject(
      rampUsers(1000) during (60 seconds),
      constantUsersPerSec(100) during (5 minutes)
    )
  ).protocols(httpProtocol)
   .assertions(
     global.responseTime.percentile(95).lt(200),
     global.successfulRequests.percent.gt(99)
   )
}
```

---

### 5. Chaos Engineering

**Scope**: Resilience testing

**Framework**: Chaos Monkey for Spring Boot

**Configuration**:
```yaml
chaos:
  monkey:
    enabled: true
    watcher:
      repository: true
      service: true
    assaults:
      level: 5
      latency-active: true
      latency-range-start: 1000
      latency-range-end: 5000
      exception-active: true
      kill-application-active: false
```

---

## Test Data Management

### Fixtures

```java
public class TestFixtures {

    public static User createUser() {
        return User.builder()
            .id(UUID.randomUUID())
            .email("test@example.com")
            .fullName("Test User")
            .kycStatus(KycStatus.APPROVED)
            .status(UserStatus.ACTIVE)
            .build();
    }

    public static Wallet createWallet(User user, String currency) {
        return Wallet.builder()
            .id(UUID.randomUUID())
            .userId(user.getId())
            .currency(currency)
            .walletType(WalletType.STANDARD)
            .status(WalletStatus.ACTIVE)
            .build();
    }
}
```

---

## CI/CD Pipeline Tests

```yaml
name: Test Pipeline

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Set up JDK 17
        uses: actions/setup-java@v2
        with:
          java-version: '17'

      - name: Run Unit Tests
        run: mvn test

      - name: Run Integration Tests
        run: mvn verify -P integration-tests

      - name: Code Coverage
        run: mvn jacoco:report
        env:
          JACOCO_MIN_COVERAGE: 80

      - name: Security Scan
        run: mvn dependency-check:check

      - name: SonarQube Analysis
        run: mvn sonar:sonar
```

---

## Coverage Goals

| Test Type | Coverage Target |
|-----------|----------------|
| **Unit Tests** | > 80% line coverage |
| **Integration Tests** | Critical paths 100% |
| **Branch Coverage** | > 70% |
| **Mutation Testing** | > 60% |

---

See [Test Data](./test-data.md) and [Performance Benchmarks](./performance-benchmarks.md) for details.
