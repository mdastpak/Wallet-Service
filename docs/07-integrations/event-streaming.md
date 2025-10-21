# Event Streaming (Kafka)

## Overview

Apache Kafka integration for event-driven architecture, audit trails, and asynchronous processing.

---

## Topics

| Topic | Partitions | Retention | Purpose |
|-------|-----------|-----------|---------|
| `wallet.transactions.v1` | 32 | 90 days | Transaction lifecycle events |
| `wallet.balances.v1` | 16 | 30 days | Balance update events |
| `wallet.rollbacks.v1` | 8 | 180 days | Rollback/compensation events |
| `wallet.audit.v1` | 32 | 365 days | Complete audit trail |
| `wallet.dlq.v1` | 4 | 30 days | Dead letter queue |

---

## Event Schema (Avro)

```json
{
  "type": "record",
  "name": "TransactionEvent",
  "namespace": "com.wallet.events",
  "fields": [
    {"name": "eventId", "type": "string"},
    {"name": "eventType", "type": "string"},
    {"name": "transactionId", "type": "string"},
    {"name": "userId", "type": "string"},
    {"name": "amount", "type": "long"},
    {"name": "currency", "type": "string"},
    {"name": "timestamp", "type": "long"},
    {"name": "metadata", "type": {"type": "map", "values": "string"}}
  ]
}
```

---

## Producer

```java
@Service
public class TransactionEventProducer {

    private final KafkaTemplate<String, TransactionEvent> kafkaTemplate;

    public void publishTransactionCreated(Transaction transaction) {
        TransactionEvent event = TransactionEvent.builder()
            .eventId(UUID.randomUUID().toString())
            .eventType("TRANSACTION_CREATED")
            .transactionId(transaction.getId().toString())
            .userId(transaction.getUserId().toString())
            .amount(transaction.getAmount())
            .currency(transaction.getCurrency())
            .timestamp(System.currentTimeMillis())
            .build();

        kafkaTemplate.send("wallet.transactions.v1",
            transaction.getUserId().toString(),  // Partition key
            event
        );

        log.info("Published TransactionCreated event: {}", event.getEventId());
    }
}
```

---

## Consumer

```java
@Service
public class BalanceAggregatorConsumer {

    @KafkaListener(
        topics = "wallet.transactions.v1",
        groupId = "balance-aggregator",
        concurrency = "3"
    )
    public void handleTransactionEvent(
        @Payload TransactionEvent event,
        @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition
    ) {
        log.info("Received event: {} from partition: {}", event.getEventId(), partition);

        try {
            updateCachedBalance(event);
        } catch (Exception e) {
            log.error("Failed to process event: {}", event.getEventId(), e);
            sendToDeadLetterQueue(event, e);
        }
    }

    private void updateCachedBalance(TransactionEvent event) {
        String key = "user:balance:" + event.getUserId();
        redisTemplate.opsForValue().increment(key, event.getAmount());
    }

    private void sendToDeadLetterQueue(TransactionEvent event, Exception e) {
        DeadLetterEvent dlq = new DeadLetterEvent(event, e.getMessage());
        kafkaTemplate.send("wallet.dlq.v1", dlq);
    }
}
```

---

## Configuration

```yaml
spring:
  kafka:
    bootstrap-servers: kafka-0.kafka:9092,kafka-1.kafka:9092,kafka-2.kafka:9092
    producer:
      acks: all
      retries: 3
      enable-idempotence: true
      compression-type: snappy
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: io.confluent.kafka.serializers.KafkaAvroSerializer
    consumer:
      group-id: wallet-service
      auto-offset-reset: earliest
      enable-auto-commit: false
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: io.confluent.kafka.serializers.KafkaAvroDeserializer
    listener:
      ack-mode: manual
```

---

## Outbox Pattern (Transactional Messaging)

```java
@Service
@Transactional
public class TransactionalPaymentService {

    public UUID createPayment(PaymentRequest request) {
        // 1. Save transaction to database
        Transaction tx = new Transaction();
        transactionRepo.save(tx);

        // 2. Save event to outbox table (same transaction)
        OutboxEvent outbox = new OutboxEvent();
        outbox.setAggregateId(tx.getId());
        outbox.setEventType("TransactionCreated");
        outbox.setPayload(serializeEvent(tx));
        outboxRepo.save(outbox);

        // 3. Transaction commits (both DB and outbox)
        return tx.getId();
    }
}

// Separate process publishes from outbox to Kafka
@Scheduled(fixedDelay = 1000)
public void publishOutboxEvents() {
    List<OutboxEvent> events = outboxRepo.findUnpublished();
    for (OutboxEvent event : events) {
        kafkaTemplate.send("wallet.transactions.v1", event.getPayload());
        event.setPublished(true);
        outboxRepo.save(event);
    }
}
```

---

## Dead Letter Queue Handling

```java
@Service
public class DeadLetterQueueProcessor {

    @KafkaListener(topics = "wallet.dlq.v1")
    public void processDLQ(DeadLetterEvent event) {
        log.warn("Processing DLQ event: {}", event);

        // Alert operations
        alertService.sendAlert("DLQ event detected", event);

        // Store for manual review
        dlqRepo.save(event);
    }
}
```

---

See [Data Flows](../01-architecture/data-flows.md) for event-driven aggregation details.
