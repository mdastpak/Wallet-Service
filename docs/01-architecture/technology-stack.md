# Technology Stack

## Overview

The Wallet Service technology stack has been carefully selected to provide enterprise-grade security, scalability, and compliance for financial operations. Each component has been chosen based on proven track record in high-volume transaction systems.

---

## Backend Framework

### Spring Boot 3.x

**Version**: 3.2.x (Spring Framework 6.x)

**Justification**:
- Industry-standard framework for enterprise Java applications
- Comprehensive security features (Spring Security)
- Excellent transaction management (Spring Data JPA)
- Built-in observability (Actuator, Micrometer)
- Strong ecosystem and community support
- Native support for OAuth 2.0 Resource Server

**Key Modules**:
```xml
<dependencies>
    <!-- Core Framework -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Data Access -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- Security -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
    </dependency>

    <!-- Redis -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>

    <!-- Kafka -->
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>

    <!-- Observability -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
</dependencies>
```

---

### Java 17 (LTS)

**Justification**:
- Long-term support until September 2029
- Performance improvements over Java 11 (G1GC enhancements, ZGC)
- Modern language features (records, sealed classes, pattern matching)
- Enhanced security features
- Strong backward compatibility

**JVM Configuration** (Production):
```bash
JAVA_OPTS="
  -Xms4g -Xmx4g
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=200
  -XX:+HeapDumpOnOutOfMemoryError
  -XX:HeapDumpPath=/var/log/wallet-service/
  -Dspring.profiles.active=production
"
```

---

## Database

### Oracle Database 26ai (Enterprise Edition)

**Justification**:
- **ACID Compliance**: Strongest consistency guarantees for financial data
- **Partitioning**: Native table partitioning for high-volume transaction tables
- **Advanced Security**: Transparent Data Encryption (TDE), Virtual Private Database (VPD)
- **High Availability**: Oracle Data Guard, Real Application Clusters (RAC)
- **Performance**: Advanced indexing, materialized views, query optimization
- **Audit**: Fine-grained auditing capabilities for compliance
- **Proven Track Record**: Used by major financial institutions globally

**Key Features Used**:

| Feature | Use Case |
|---------|----------|
| **Range Partitioning** | Partition transactions table by month/year |
| **Bitmap Indexes** | Fast queries on status/type columns |
| **Materialized Views** | Precomputed aggregations for reporting |
| **TDE** | Encrypt PII columns at rest |
| **Data Guard** | Synchronous replication for HA |
| **Flashback** | Point-in-time recovery for audit |
| **Advisory Locks** | Distributed locking for critical sections |

**Connection Pooling** (HikariCP):
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 50
      minimum-idle: 10
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      leak-detection-threshold: 60000
```

---

## Caching & Distributed Locking

### Redis Cluster 7.x

**Justification**:
- **High Performance**: Sub-millisecond latency for cache operations
- **Distributed Locking**: SET NX PX for idempotency enforcement
- **Pub/Sub**: Cache invalidation notifications
- **Clustering**: Horizontal scaling and automatic sharding
- **Persistence**: AOF + RDB for durability
- **High Availability**: Redis Sentinel for automatic failover

**Use Cases**:

| Use Case | Data Structure | TTL |
|----------|---------------|-----|
| **Idempotency Locks** | String (SET NX PX) | 5min - 24h (configurable) |
| **Cached Balances** | Hash | 5 minutes |
| **Session Storage** | Hash | 30 minutes |
| **Rate Limiting** | String (INCR) | 1 minute |
| **Distributed Locks** | String (Redlock) | 30 seconds |

**Configuration**:
```yaml
spring:
  redis:
    cluster:
      nodes:
        - redis-node-1:6379
        - redis-node-2:6379
        - redis-node-3:6379
      max-redirects: 3
    timeout: 2000ms
    lettuce:
      pool:
        max-active: 50
        max-idle: 20
        min-idle: 5
```

**Idempotency Lock Pattern**:
```java
// Acquire lock with TTL
Boolean acquired = redisTemplate.opsForValue()
    .setIfAbsent(
        "idempotency:" + key,
        "locked",
        Duration.ofMinutes(ttlMinutes)
    );
```

---

## Event Streaming

### Apache Kafka 3.x

**Justification**:
- **Event Sourcing**: Immutable log for all domain events
- **Audit Trail**: Complete history of all financial operations
- **Scalability**: Handles millions of events per second
- **Durability**: Replication and persistence guarantees
- **Stream Processing**: Real-time aggregation with Kafka Streams
- **Integration**: Connects to external systems (analytics, data warehouse)

**Topics**:

| Topic | Partitions | Retention | Purpose |
|-------|-----------|-----------|---------|
| `wallet.transactions.v1` | 32 | 90 days | Transaction lifecycle events |
| `wallet.balances.v1` | 16 | 30 days | Balance update events |
| `wallet.rollbacks.v1` | 8 | 180 days | Rollback/compensation events |
| `wallet.audit.v1` | 32 | 365 days | Complete audit trail |
| `wallet.dlq.v1` | 4 | 30 days | Dead letter queue |

**Event Schema** (Avro):
```json
{
  "type": "record",
  "name": "TransactionEvent",
  "namespace": "com.wallet.events",
  "fields": [
    {"name": "eventId", "type": "string"},
    {"name": "transactionId", "type": "string"},
    {"name": "eventType", "type": "string"},
    {"name": "timestamp", "type": "long"},
    {"name": "userId", "type": "string"},
    {"name": "amount", "type": "long"},
    {"name": "currency", "type": "string"},
    {"name": "metadata", "type": "map", "values": "string"}
  ]
}
```

**Producer Configuration**:
```yaml
spring:
  kafka:
    producer:
      acks: all
      retries: 3
      enable-idempotence: true
      compression-type: snappy
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: io.confluent.kafka.serializers.KafkaAvroSerializer
```

---

## Authentication & Authorization

### OAuth 2.0 + mTLS

**OAuth 2.0 (B2C)**:

**Justification**:
- Industry-standard protocol
- Stateless JWT tokens
- Scoped access control
- Supports multiple grant types (authorization code, client credentials)

**Flows**:
- **Web/Mobile**: Authorization Code Flow with PKCE
- **Server-to-Server**: Client Credentials Flow

**JWT Claims**:
```json
{
  "sub": "user-123",
  "iss": "https://auth.wallet-service.com",
  "aud": "wallet-api",
  "exp": 1698765432,
  "iat": 1698761832,
  "scope": "wallet:read wallet:write",
  "business_id": "business-456",
  "roles": ["USER"]
}
```

**mTLS (B2B)**:

**Justification**:
- Mutual authentication for machine-to-machine
- Certificate-based identity verification
- No shared secrets
- Strong cryptographic guarantees

**Certificate Validation**:
```java
@Configuration
public class MTlsConfig {
    @Bean
    public TomcatServletWebServerFactory servletContainer() {
        TomcatServletWebServerFactory tomcat = new TomcatServletWebServerFactory();
        tomcat.addConnectorCustomizers(connector -> {
            connector.setSecure(true);
            connector.setScheme("https");
            Http11NioProtocol protocol = (Http11NioProtocol) connector.getProtocolHandler();
            protocol.setSSLEnabled(true);
            protocol.setClientAuth("require");
        });
        return tomcat;
    }
}
```

**RBAC Roles**:

| Role | Permissions |
|------|-------------|
| `USER` | Read own wallets, create payments |
| `BUSINESS_ADMIN` | Manage business users, view business aggregates |
| `FINANCE_ADMIN` | Process rollbacks, apply discounts |
| `SYSTEM_ADMIN` | All operations, system configuration |

---

## Resilience & Fault Tolerance

### Resilience4j

**Justification**:
- Lightweight resilience library for Java
- Circuit breaker, retry, rate limiter, bulkhead patterns
- Excellent Spring Boot integration
- Metrics and monitoring support

**Circuit Breaker Configuration**:
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
```

**Retry Configuration**:
```yaml
resilience4j:
  retry:
    instances:
      paymentGateway:
        max-attempts: 3
        wait-duration: 1s
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - java.net.SocketTimeoutException
          - org.springframework.web.client.ResourceAccessException
```

---

## Database Migration

### Flyway

**Justification**:
- Version-controlled database schema
- Repeatable migrations
- Support for Oracle-specific features
- Integration with Spring Boot

**Migration Naming**:
```
V1__initial_schema.sql
V2__add_idempotency_table.sql
V3__add_discount_codes.sql
V4__partition_transactions.sql
```

**Configuration**:
```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    validate-on-migrate: true
    clean-disabled: true
```

---

## API Documentation

### OpenAPI 3.0 (Springdoc)

**Justification**:
- Auto-generated API documentation
- Interactive Swagger UI
- Client SDK generation
- Contract-first development support

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.2.0</version>
</dependency>
```

---

## Monitoring & Observability

### Prometheus + Grafana

**Prometheus**:
- Metrics collection and storage
- Powerful query language (PromQL)
- Alerting rules

**Grafana**:
- Visualization dashboards
- Multi-datasource support
- Alerting and notifications

**Custom Metrics**:
```java
@Component
public class PaymentMetrics {
    private final Counter paymentCounter;
    private final Timer paymentTimer;

    public PaymentMetrics(MeterRegistry registry) {
        this.paymentCounter = Counter.builder("wallet.payments.total")
            .tag("type", "payment")
            .register(registry);

        this.paymentTimer = Timer.builder("wallet.payments.duration")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
    }
}
```

### OpenTelemetry

**Distributed Tracing**:
- Trace payment flows across services
- Performance bottleneck identification
- Integration with Jaeger/Zipkin

---

## Containerization & Orchestration

### Docker

**Base Image**:
```dockerfile
FROM eclipse-temurin:17-jre-alpine
VOLUME /tmp
COPY target/wallet-service.jar app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

### Kubernetes

**Deployment Strategy**:
- Rolling updates (zero downtime)
- Horizontal Pod Autoscaler (HPA)
- Resource limits and requests
- Health checks (liveness/readiness)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wallet-service
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: wallet-service
        image: wallet-service:1.0.0
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
```

---

## Security

### Encryption

**At Rest**: AES-256-GCM
**In Transit**: TLS 1.3
**Key Management**: AWS KMS / HashiCorp Vault

**Example (JPA Attribute Encryption)**:
```java
@Converter
public class EncryptedStringConverter implements AttributeConverter<String, String> {

    @Autowired
    private KmsService kmsService;

    @Override
    public String convertToDatabaseColumn(String attribute) {
        return kmsService.encrypt(attribute);
    }

    @Override
    public String convertToEntityAttribute(String dbData) {
        return kmsService.decrypt(dbData);
    }
}
```

---

## Testing

### JUnit 5 + Mockito

**Unit Testing**:
```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <scope>test</scope>
</dependency>
```

### Testcontainers

**Integration Testing** with real Oracle, Redis, Kafka:
```java
@Testcontainers
@SpringBootTest
public class PaymentServiceIntegrationTest {

    @Container
    static OracleContainer oracle = new OracleContainer("gvenzl/oracle-xe:21-slim");

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);

    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0")
    );
}
```

---

## Summary Table

| Category | Technology | Version | Purpose |
|----------|-----------|---------|---------|
| **Language** | Java | 17 LTS | Application runtime |
| **Framework** | Spring Boot | 3.2.x | Application framework |
| **Database** | Oracle Database | 26ai Enterprise | ACID transactional store |
| **Cache** | Redis Cluster | 7.x | Distributed cache/locks |
| **Messaging** | Apache Kafka | 3.x | Event streaming |
| **Auth** | OAuth 2.0 + mTLS | - | Authentication/authorization |
| **Resilience** | Resilience4j | 2.x | Circuit breaker, retry |
| **Migration** | Flyway | 9.x | Database versioning |
| **API Docs** | Springdoc OpenAPI | 2.x | API documentation |
| **Monitoring** | Prometheus + Grafana | - | Metrics & dashboards |
| **Tracing** | OpenTelemetry | 1.x | Distributed tracing |
| **Container** | Docker | 24.x | Containerization |
| **Orchestration** | Kubernetes | 1.28+ | Container orchestration |
| **Testing** | JUnit 5 + Testcontainers | 5.x / 1.19.x | Unit & integration tests |

---

## Next Steps

- Review [Deployment Architecture](./deployment-architecture.md) for Kubernetes setup
- See [Data Flows](./data-flows.md) for component interactions
- Explore [Security Architecture](../04-security-compliance/security-architecture.md) for detailed security controls
