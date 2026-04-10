# Production Spring Boot Service

## Service Structure

```
banking-payment-service/
├── src/
│   ├── main/
│   │   ├── java/com/bank/payment/
│   │   │   ├── PaymentServiceApplication.java
│   │   │   ├── config/
│   │   │   │   ├── SecurityConfig.java
│   │   │   │   ├── DatabaseConfig.java
│   │   │   │   ├── KafkaConfig.java
│   │   │   │   └── OpenApiConfig.java
│   │   │   ├── controller/
│   │   │   │   ├── PaymentController.java
│   │   │   │   ├── AccountController.java
│   │   │   │   └── HealthController.java
│   │   │   ├── service/
│   │   │   │   ├── PaymentService.java
│   │   │   │   ├── AccountService.java
│   │   │   │   └── NotificationService.java
│   │   │   ├── repository/
│   │   │   │   ├── PaymentRepository.java
│   │   │   │   └── AccountRepository.java
│   │   │   ├── model/
│   │   │   │   ├── Payment.java
│   │   │   │   ├── Account.java
│   │   │   │   └── AuditLog.java
│   │   │   ├── dto/
│   │   │   │   ├── PaymentRequest.java
│   │   │   │   ├── PaymentResponse.java
│   │   │   │   └── ErrorResponse.java
│   │   │   ├── exception/
│   │   │   │   ├── GlobalExceptionHandler.java
│   │   │   │   ├── PaymentException.java
│   │   │   │   └── InsufficientFundsException.java
│   │   │   └── listener/
│   │   │       └── PaymentEventListener.java
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       ├── application-prod.yml
│   │       └── db/migration/
│   │           └── V1__initial_schema.sql
│   └── test/
│       └── java/com/bank/payment/
├── pom.xml
├── Dockerfile
└── docker-compose.yml
```

## Application Entry Point

```java
@SpringBootApplication
@EnableJpaRepositories
@EnableScheduling
public class PaymentServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(PaymentServiceApplication.class, args);
    }
}
```

## Configuration

```yaml
# application.yml
spring:
  application:
    name: banking-payment-service

  datasource:
    url: ${DATABASE_URL:jdbc:postgresql://localhost:5432/banking}
    username: ${DATABASE_USERNAME:bank_user}
    password: ${DATABASE_PASSWORD:secret}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      idle-timeout: 300000
      max-lifetime: 1800000
      connection-timeout: 30000

  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    open-in-view: false
    properties:
      hibernate:
        format_sql: true
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true

  flyway:
    enabled: true
    locations: classpath:db/migration

  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    consumer:
      group-id: payment-service
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer

server:
  port: ${PORT:8080}
  error:
    include-message: never
    include-binding-errors: never
    include-stacktrace: never

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always

logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] [%X{correlationId}] %-5level %logger{36} - %msg%n"
  level:
    com.bank.payment: INFO
    org.hibernate.SQL: DEBUG
```

## Controller

```java
@RestController
@RequestMapping("/v1/payments")
@Validated
@RequiredArgsConstructor
@Slf4j
public class PaymentController {
    private final PaymentService paymentService;

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ResponseEntity<PaymentResponse> createPayment(
            @Valid @RequestBody PaymentRequest request,
            @RequestHeader(value = "X-Correlation-ID", required = false) String correlationId) {

        log.info("Creating payment for account {}", request.fromAccount());

        Payment payment = paymentService.processPayment(
            PaymentCommand.builder()
                .fromAccount(request.fromAccount())
                .toAccount(request.toAccount())
                .amount(request.amount())
                .currency(request.currency())
                .reference(request.reference())
                .build()
        );

        return ResponseEntity
            .created(URI.create("/v1/payments/" + payment.getId()))
            .header("X-Correlation-ID", correlationId)
            .body(PaymentResponse.from(payment));
    }

    @GetMapping("/{id}")
    public ResponseEntity<PaymentResponse> getPayment(@PathVariable String id) {
        Payment payment = paymentService.getPayment(id)
            .orElseThrow(() -> new PaymentNotFoundException(id));

        return ResponseEntity.ok(PaymentResponse.from(payment));
    }

    @GetMapping("/{accountId}/transactions")
    public ResponseEntity<Page<PaymentResponse>> getTransactions(
            @PathVariable String accountId,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "50") int size) {

        Pageable pageable = PageRequest.of(page, size, Sort.by("createdAt").descending());
        Page<Payment> payments = paymentService.getTransactions(accountId, pageable);

        return ResponseEntity.ok(payments.map(PaymentResponse::from));
    }
}
```

## DTOs with Validation

```java
public record PaymentRequest(
    @NotBlank(message = "From account is required")
    @Size(min = 10, max = 34, message = "Invalid account number format")
    String fromAccount,

    @NotBlank(message = "To account is required")
    @Size(min = 10, max = 34, message = "Invalid account number format")
    String toAccount,

    @NotNull(message = "Amount is required")
    @DecimalMin(value = "0.01", message = "Amount must be greater than 0")
    @DecimalMax(value = "1000000.00", message = "Amount exceeds maximum")
    BigDecimal amount,

    @NotBlank(message = "Currency is required")
    @Pattern(regexp = "^[A-Z]{3}$", message = "Invalid currency code")
    String currency,

    @Size(max = 140, message = "Reference too long")
    String reference
) {}

public record PaymentResponse(
    String id,
    String fromAccount,
    String toAccount,
    BigDecimal amount,
    String currency,
    String status,
    String reference,
    Instant createdAt
) {
    public static PaymentResponse from(Payment payment) {
        return new PaymentResponse(
            payment.getId(),
            payment.getFromAccount(),
            payment.getToAccount(),
            payment.getAmount(),
            payment.getCurrency(),
            payment.getStatus().name(),
            payment.getReference(),
            payment.getCreatedAt()
        );
    }
}
```

## Service Layer

```java
@Service
@RequiredArgsConstructor
@Slf4j
@Transactional
public class PaymentService {
    private final PaymentRepository paymentRepo;
    private final AccountRepository accountRepo;
    private final ApplicationEventPublisher eventPublisher;

    public Payment processPayment(PaymentCommand command) {
        // Validate accounts exist
        Account fromAccount = accountRepo.findById(command.fromAccount())
            .orElseThrow(() -> new AccountNotFoundException(command.fromAccount()));
        Account toAccount = accountRepo.findById(command.toAccount())
            .orElseThrow(() -> new AccountNotFoundException(command.toAccount()));

        // Check sufficient funds
        if (fromAccount.getBalance().compareTo(command.amount()) < 0) {
            throw new InsufficientFundsException(fromAccount.getId(), command.amount());
        }

        // Create payment
        Payment payment = Payment.builder()
            .fromAccount(command.fromAccount())
            .toAccount(command.toAccount())
            .amount(command.amount())
            .currency(command.currency())
            .reference(command.reference())
            .status(PaymentStatus.PENDING)
            .createdAt(Instant.now())
            .build();

        payment = paymentRepo.save(payment);

        // Update account balances
        fromAccount.setBalance(fromAccount.getBalance().subtract(command.amount()));
        toAccount.setBalance(toAccount.getBalance().add(command.amount()));
        accountRepo.saveAll(List.of(fromAccount, toAccount));

        // Update payment status
        payment.setStatus(PaymentStatus.COMPLETED);
        payment = paymentRepo.save(payment);

        // Publish event for async processing
        eventPublisher.publishEvent(new PaymentCompletedEvent(payment));

        return payment;
    }

    @Transactional(readOnly = true)
    public Optional<Payment> getPayment(String id) {
        return paymentRepo.findById(id);
    }

    @Transactional(readOnly = true)
    public Page<Payment> getTransactions(String accountId, Pageable pageable) {
        return paymentRepo.findByFromAccountOrToAccount(accountId, accountId, pageable);
    }
}
```

## Exception Handling

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(InsufficientFundsException.class)
    public ResponseEntity<ErrorResponse> handleInsufficientFunds(InsufficientFundsException ex) {
        ErrorResponse error = ErrorResponse.builder()
            .code("INSUFFICIENT_FUNDS")
            .message(ex.getMessage())
            .timestamp(Instant.now())
            .build();

        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY).body(error);
    }

    @ExceptionHandler(PaymentNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(PaymentNotFoundException ex) {
        ErrorResponse error = ErrorResponse.builder()
            .code("NOT_FOUND")
            .message(ex.getMessage())
            .timestamp(Instant.now())
            .build();

        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        List<FieldError> fieldErrors = ex.getBindingResult().getFieldErrors().stream()
            .map(fe -> new FieldError(fe.getField(), fe.getDefaultMessage()))
            .toList();

        ErrorResponse error = ErrorResponse.builder()
            .code("VALIDATION_ERROR")
            .message("Request validation failed")
            .details(fieldErrors)
            .timestamp(Instant.now())
            .build();

        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        log.error("Unexpected error", ex);

        ErrorResponse error = ErrorResponse.builder()
            .code("INTERNAL_ERROR")
            .message("An unexpected error occurred")
            .timestamp(Instant.now())
            .build();

        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

## Health Checks

```java
@RestController
@RequestMapping("/health")
@RequiredArgsConstructor
public class HealthController {
    private final DataSource dataSource;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    @GetMapping("/live")
    public ResponseEntity<Map<String, String>> liveness() {
        return ResponseEntity.ok(Map.of("status", "alive"));
    }

    @GetMapping("/ready")
    public ResponseEntity<Map<String, Object>> readiness() {
        Map<String, Object> checks = new HashMap<>();

        // Database check
        try (Connection conn = dataSource.getConnection()) {
            checks.put("database", "ok");
        } catch (SQLException e) {
            checks.put("database", "error: " + e.getMessage());
        }

        // Kafka check
        try {
            kafkaTemplate.send("health-check", "ping").get(5, TimeUnit.SECONDS);
            checks.put("kafka", "ok");
        } catch (Exception e) {
            checks.put("kafka", "error: " + e.getMessage());
        }

        boolean allOk = checks.values().stream().allMatch(v -> v.equals("ok"));
        Map<String, Object> response = Map.of(
            "status", allOk ? "ready" : "not_ready",
            "checks", checks
        );

        return ResponseEntity.ok(response);
    }
}
```

## Docker Deployment

```dockerfile
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

# Copy JAR from build
COPY target/*.jar app.jar

# Non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 8080

ENTRYPOINT ["java", \
    "-XX:+UseG1GC", \
    "-XX:MaxGCPauseMillis=100", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-jar", "app.jar"]
```

## Common Mistakes

1. **`open-in-view: true`**: Lazy loading in view causes N+1 queries
2. **Not using `@Transactional(readOnly = true)`**: Unnecessary write locks on reads
3. **Swallowing exceptions in `@ExceptionHandler`**: Not logging root cause
4. **Using `H2` for tests**: Different SQL dialect from PostgreSQL
5. **Not configuring connection pool**: Default pool size too small for production
6. **No graceful shutdown**: Active requests terminated on SIGTERM

## Interview Questions

1. How do you configure Spring Boot for a zero-downtime deployment?
2. Design a payment service with audit logging and event publishing.
3. How do you handle `@Transactional` boundaries in a service method?
4. What is the difference between `@ControllerAdvice` and `@ExceptionHandler`?
5. How do you configure health checks that verify database and Kafka connectivity?

## Cross-References

- `./dependency-injection.md` - Spring DI patterns
- `./testing.md` - Testing Spring Boot services
- `./microservices.md` - Spring Cloud patterns
- `../api-design/README.md` - REST API design
- `../microservices/README.md` - Microservice architecture
