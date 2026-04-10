# Java Microservices with Spring Cloud

## Spring Cloud Components

```
Spring Cloud provides patterns for microservices:

Spring Cloud Gateway    - API gateway and routing
Spring Cloud Config     - Centralized configuration
Spring Cloud Circuit    - Circuit breaker (Resilience4j)
Spring Cloud OpenFeign  - Declarative REST clients
Spring Cloud Stream     - Event streaming (Kafka/RabbitMQ)
Spring Cloud Sleuth     - Distributed tracing
Eureka                  - Service discovery (legacy, use Kubernetes instead)
```

## Service-to-Service Communication

### OpenFeign Client

```java
// Declarative REST client
@FeignClient(
    name = "account-service",
    url = "${services.account-service.url}",
    configuration = AccountServiceClientConfig.class
)
public interface AccountServiceClient {
    @GetMapping("/v1/accounts/{accountId}")
    ResponseEntity<AccountResponse> getAccount(@PathVariable String accountId);

    @PostMapping("/v1/accounts/{accountId}/lock")
    ResponseEntity<Void> lockAccount(@PathVariable String accountId);

    @PostMapping("/v1/accounts/{accountId}/debit")
    ResponseEntity<Void> debit(@PathVariable String accountId, @RequestBody DebitRequest request);
}

// Usage
@Service
@RequiredArgsConstructor
public class PaymentService {
    private final AccountServiceClient accountClient;

    public Payment processPayment(PaymentCommand cmd) {
        // Call account service
        AccountResponse account = accountClient.getAccount(cmd.fromAccount()).getBody();

        if (account.getBalance().compareTo(cmd.amount()) < 0) {
            throw new InsufficientFundsException(cmd.fromAccount());
        }

        accountClient.debit(cmd.fromAccount(), new DebitRequest(cmd.amount()));

        // Process payment...
    }
}
```

### Circuit Breaker with Resilience4j

```java
@Service
@RequiredArgsConstructor
public class PaymentGatewayClient {
    private final RestTemplate restTemplate;

    @CircuitBreaker(name = "paymentGateway", fallbackMethod = "gatewayFallback")
    @Retry(name = "paymentGateway")
    @TimeLimiter(name = "paymentGateway")
    public CompletableFuture<PaymentResult> initiatePayment(PaymentRequest request) {
        return CompletableFuture.supplyAsync(() -> {
            ResponseEntity<PaymentResult> response = restTemplate.postForEntity(
                gatewayUrl + "/payments", request, PaymentResult.class);
            return response.getBody();
        });
    }

    // Fallback when circuit is open
    public CompletableFuture<PaymentResult> gatewayFallback(PaymentRequest request, Throwable t) {
        log.warn("Payment gateway unavailable, queuing payment: {}", request);
        paymentQueue.enqueue(request);
        return CompletableFuture.completedFuture(
            new PaymentResult("QUEUED", "Payment queued for later processing"));
    }
}
```

### Configuration

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      paymentGateway:
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        sliding-window-size: 10
        minimum-number-of-calls: 5
        automatic-transition-from-open-to-half-open-enabled: true

  retry:
    instances:
      paymentGateway:
        max-attempts: 3
        wait-duration: 1s
        retry-exceptions:
          - org.springframework.web.client.ResourceAccessException
          - java.net.SocketTimeoutException

  timelimiter:
    instances:
      paymentGateway:
        timeout-duration: 10s
```

## Event-Driven Communication

### Spring Cloud Stream with Kafka

```java
// Payment event publisher
@Component
@RequiredArgsConstructor
public class PaymentEventPublisher {
    private final StreamBridge streamBridge;

    public void publishPaymentCompleted(Payment payment) {
        streamBridge.send("paymentCompleted-out-0",
            MessageBuilder.withPayload(payment)
                .setHeader("partitionKey", payment.getFromAccount())
                .build());
    }
}

// Payment event consumer
@Component
@Slf4j
public class PaymentEventConsumer {
    @Bean
    public Consumer<PaymentCompletedEvent> handlePaymentCompleted() {
        return event -> {
            log.info("Payment completed: {}", event.getPaymentId());
            // Update local read model
            // Send notification
            // Update analytics
        };
    }
}
```

```yaml
spring:
  cloud:
    stream:
      bindings:
        paymentCompleted-out-0:
          destination: payments.completed
          content-type: application/json
        handlePaymentCompleted-in-0:
          destination: payments.completed
          group: notification-service
          content-type: application/json
      kafka:
        binder:
          brokers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
```

## Distributed Tracing

```yaml
# Micrometer Tracing (replaces Spring Cloud Sleuth)
management:
  tracing:
    sampling:
      probability: 1.0  # Sample all traces (reduce in production)
  zipkin:
    tracing:
      endpoint: ${ZIPKIN_URL:http://zipkin:9411}/api/v2/spans
```

```java
// Propagate trace ID in HTTP calls
@FeignClient(name = "account-service")
public interface AccountServiceClient {
    // Trace ID automatically propagated via Spring Cloud
    @GetMapping("/v1/accounts/{id}")
    AccountResponse getAccount(@PathVariable String id);
}

// Log correlation ID
logging:
  pattern:
    console: "%d{HH:mm:ss.SSS} [%thread] [%X{traceId},%X{spanId}] %-5level %logger - %msg%n"
```

## Saga Pattern

```java
@Service
@RequiredArgsConstructor
public class TransferSaga {
    private final AccountServiceClient accountClient;
    private final PaymentRepository paymentRepo;
    private final CompensatingActionService compensator;

    @Transactional
    public TransferResult executeTransfer(TransferCommand cmd) {
        List<Compensation> compensations = new ArrayList<>();

        try {
            // Step 1: Debit source
            accountClient.debit(cmd.fromAccount(), new DebitRequest(cmd.amount()));
            compensations.add(() -> accountClient.credit(cmd.fromAccount(), new CreditRequest(cmd.amount())));

            // Step 2: Credit destination
            accountClient.credit(cmd.toAccount(), new CreditRequest(cmd.amount()));
            compensations.add(() -> accountClient.debit(cmd.toAccount(), new DebitRequest(cmd.amount())));

            // Step 3: Record payment
            Payment payment = paymentRepo.save(Payment.from(cmd));

            return TransferResult.success(payment.getId());

        } catch (Exception e) {
            // Compensate: undo in reverse order
            Collections.reverse(compensations);
            for (Compensation comp : compensations) {
                try {
                    comp.execute();
                } catch (Exception compError) {
                    log.error("Compensation failed: {}", compError.getMessage(), compError);
                    // Alert oncall - manual intervention needed
                    alertOnCallTeam(compError);
                }
            }
            throw new TransferFailedException(cmd, e);
        }
    }

    @FunctionalInterface
    interface Compensation {
        void execute();
    }
}
```

## API Gateway

```yaml
# Spring Cloud Gateway configuration
spring:
  cloud:
    gateway:
      routes:
        - id: payment-service
          uri: ${PAYMENT_SERVICE_URL:http://payment-service:8080}
          predicates:
            - Path=/v1/payments/**
          filters:
            - StripPrefix=0
            - name: CircuitBreaker
              args:
                name: paymentService
                fallbackUri: forward:/fallback/payment

        - id: account-service
          uri: ${ACCOUNT_SERVICE_URL:http://account-service:8080}
          predicates:
            - Path=/v1/accounts/**

        - id: ai-service
          uri: ${AI_SERVICE_URL:http://ai-service:8080}
          predicates:
            - Path=/v1/ai/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
```

## Common Mistakes

1. **Sync calls without circuit breaker**: Cascading failures across services
2. **No distributed tracing**: Cannot debug cross-service requests
3. **Shared database between services**: Violates microservice boundaries
4. **No saga compensation**: Partial failures leave data inconsistent
5. **Eureka in Kubernetes**: Kubernetes already provides service discovery
6. **Not testing service failures**: No chaos engineering or failure injection

## Interview Questions

1. How do you implement the saga pattern for a transfer spanning 3 services?
2. Design a circuit breaker for an external payment gateway.
3. How do you trace a request across 5 microservices?
4. What is the difference between Spring Cloud Gateway and Kubernetes Ingress?
5. How do you handle eventual consistency between services?

## Cross-References

- `./spring-boot.md` - Spring Boot service setup
- `./dependency-injection.md` - Injecting service clients
- `../microservices/README.md` - Microservice architecture patterns
- `../distributed-systems/README.md` - Distributed transactions
- `../resilience-patterns/README.md` - Circuit breaker patterns
