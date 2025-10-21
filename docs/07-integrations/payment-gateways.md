# Payment Gateway Integration

## Overview

Integration patterns for external payment gateways with resilience, retry logic, and webhook handling.

---

## Gateway Client Implementation

```java
@Service
public class PaymentGatewayClient {

    private final WebClient webClient;
    private final CircuitBreaker circuitBreaker;

    public PaymentGatewayClient(WebClient.Builder builder,
                               CircuitBreakerRegistry registry) {
        this.webClient = builder
            .baseUrl("${payment.gateway.url}")
            .build();

        this.circuitBreaker = registry.circuitBreaker("paymentGateway");
    }

    @Retry(name = "paymentGateway")
    @CircuitBreaker(name = "paymentGateway", fallbackMethod = "processPaymentFallback")
    public GatewayResponse processPayment(PaymentRequest request) {
        return webClient.post()
            .uri("/process")
            .header("Authorization", "Bearer " + apiKey)
            .bodyValue(request)
            .retrieve()
            .bodyToMono(GatewayResponse.class)
            .timeout(Duration.ofSeconds(5))
            .block();
    }

    private GatewayResponse processPaymentFallback(PaymentRequest request, Exception e) {
        log.error("Payment gateway unavailable, marking for manual review", e);
        return GatewayResponse.manualReview(request);
    }
}
```

---

## Resilience Configuration

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentGateway:
        sliding-window-size: 100
        failure-rate-threshold: 50
        wait-duration-in-open-state: 60s
        permitted-number-of-calls-in-half-open-state: 10
        automatic-transition-from-open-to-half-open-enabled: true

  retry:
    instances:
      paymentGateway:
        max-attempts: 3
        wait-duration: 1s
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - java.net.SocketTimeoutException
          - org.springframework.web.client.ResourceAccessException
        ignore-exceptions:
          - com.wallet.PaymentDeclinedException
```

---

## Webhook Handling

**Endpoint**: `POST /webhooks/payment-gateway`

```java
@RestController
@RequestMapping("/webhooks")
public class PaymentWebhookController {

    @PostMapping("/payment-gateway")
    public ResponseEntity<Void> handleWebhook(
        @RequestBody GatewayWebhookPayload payload,
        @RequestHeader("X-Signature") String signature
    ) {
        // Verify webhook signature
        if (!webhookValidator.isValid(payload, signature)) {
            return ResponseEntity.status(401).build();
        }

        // Process webhook asynchronously
        webhookProcessor.process(payload);

        return ResponseEntity.ok().build();
    }
}

@Service
public class WebhookProcessor {

    @Async
    @Transactional
    public void process(GatewayWebhookPayload payload) {
        switch (payload.getEventType()) {
            case "payment.succeeded":
                handlePaymentSucceeded(payload);
                break;
            case "payment.failed":
                handlePaymentFailed(payload);
                break;
            case "refund.completed":
                handleRefundCompleted(payload);
                break;
        }
    }

    private void handlePaymentSucceeded(GatewayWebhookPayload payload) {
        Transaction tx = transactionRepo.findByGatewayId(payload.getTransactionId())
            .orElseThrow();

        tx.setStatus(TransactionStatus.CONFIRMED);
        transactionRepo.save(tx);

        // Emit event
        kafkaTemplate.send("wallet.transactions.v1",
            new TransactionConfirmedEvent(tx.getId()));
    }
}
```

---

## Idempotent Webhook Processing

```java
@Service
public class WebhookIdempotencyService {

    public boolean isProcessed(String webhookId) {
        return redisTemplate.opsForValue()
            .setIfAbsent("webhook:" + webhookId, "processed", Duration.ofHours(24));
    }
}
```

---

## Gateway Abstraction

Support multiple payment gateways:

```java
public interface PaymentGatewayProvider {
    GatewayResponse process(PaymentRequest request);
    RefundResponse refund(String transactionId);
    String getProviderId();
}

@Service
public class StripeGatewayProvider implements PaymentGatewayProvider {
    @Override
    public GatewayResponse process(PaymentRequest request) {
        // Stripe-specific implementation
    }
}

@Service
public class PayPalGatewayProvider implements PaymentGatewayProvider {
    @Override
    public GatewayResponse process(PaymentRequest request) {
        // PayPal-specific implementation
    }
}

@Service
public class PaymentGatewayRouter {

    private final Map<String, PaymentGatewayProvider> providers;

    public GatewayResponse route(PaymentRequest request) {
        String providerId = request.getPreferredGateway();
        PaymentGatewayProvider provider = providers.get(providerId);
        return provider.process(request);
    }
}
```

---

## Testing

**Mock Gateway for Integration Tests**:

```java
@RestController
@Profile("test")
public class MockPaymentGateway {

    @PostMapping("/process")
    public GatewayResponse processPayment(@RequestBody PaymentRequest request) {
        return GatewayResponse.builder()
            .transactionId(UUID.randomUUID().toString())
            .status("SUCCESS")
            .build();
    }
}
```

---

See [Data Flows](../01-architecture/data-flows.md) for complete payment processing sequence.
