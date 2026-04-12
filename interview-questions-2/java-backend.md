# Java Backend Interview Questions — 20 Questions with Full Answers

## Overview
| Detail | Value |
|--------|-------|
| Topic | Java Backend (Spring Boot, Dependency Injection, Microservices, JPA, Testing, Performance) |
| Questions | 20 (5 Must-Know, 10 Medium, 5 Advanced) |
| Source Files | backend-engineering/java/ (6 files: README, dependency-injection, microservices, performance, spring-boot, testing) |
| Citi Relevance | Java is one of Citi's four backend languages. The bank has massive existing Java infrastructure. Spring Boot + JPA are ubiquitous in enterprise Java. |

---

## Difficulty Legend
- 🔵 **Must-Know** — Foundational. Every candidate should know these.
- 🟡 **Medium** — Core competency. What you'll actually be tested on.
- 🔴 **Advanced** — Differentiator. Shows principal-engineer depth.

---

### Q1: What is Spring Boot auto-configuration and how does it work under the hood?

**Strong Answer:**

Spring Boot auto-configuration is a mechanism that automatically configures your Spring application based on the JAR dependencies present on the classpath and the beans you have defined. It eliminates the need for manual XML or Java-based configuration for common setups.

Under the hood, auto-configuration is powered by the `@EnableAutoConfiguration` annotation (included in `@SpringBootApplication`). This annotation triggers `AutoConfigurationImportSelector`, which reads the `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` file from all Spring Boot starter JARs. Each auto-configuration class is guarded by conditional annotations like `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`, and `@ConditionalOnWebApplication`.

For example, when `spring-boot-starter-data-jpa` is on the classpath, `DataSourceAutoConfiguration` checks if a `DataSource` bean already exists. If not, it creates one using the `spring.datasource.*` properties. Similarly, `HibernateJpaAutoConfiguration` configures the `EntityManagerFactory` and transaction manager.

In a banking context, this means a payment service can be bootstrapped with minimal configuration — Spring Boot detects PostgreSQL on the classpath, reads `spring.datasource.url` from environment variables, configures HikariCP connection pooling, and sets up JPA with Hibernate — all without explicit bean definitions.

You can exclude specific auto-configurations using `@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})` or via `spring.autoconfigure.exclude` in `application.yml`. You can also debug what auto-configurations are applied using the `--debug` flag or the Actuator `/actuator/conditions` endpoint.

**Key Points to Hit:**
- [ ] `@EnableAutoConfiguration` and `AutoConfigurationImportSelector`
- [ ] Conditional annotations (`@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`)
- [ ] Auto-configuration imports file in `META-INF/spring/`
- [ ] Starter dependencies pull in auto-configuration classes
- [ ] Ability to exclude or override auto-configurations

**Follow-Up Questions:**
1. How would you debug which auto-configurations are being applied?
2. How do you override an auto-configured bean with your own implementation?
3. What happens when two auto-configuration classes conflict?

**Source:** `backend-engineering/java/spring-boot.md`, `backend-engineering/java/dependency-injection.md`

---

### Q2: Explain the difference between constructor injection, setter injection, and field injection. Which is preferred and why?

**Strong Answer:**

Spring supports three primary injection methods, each with distinct trade-offs in a production banking environment.

**Constructor Injection** is the recommended approach. Dependencies are declared as `final` fields and passed through the constructor. This makes dependencies explicit, immutable, and visible in the class signature. It also enables straightforward unit testing — you can instantiate the class with mock dependencies without needing the Spring container. Lombok's `@RequiredArgsConstructor` eliminates the boilerplate of writing the constructor manually.

```java
@Service
@RequiredArgsConstructor
public class PaymentService {
    private final PaymentRepository paymentRepo;
    private final AccountRepository accountRepo;
    private final NotificationService notificationService;
}
```

**Setter Injection** uses `@Autowired` on setter methods. It is appropriate only for optional dependencies where the component can function without the dependency. You mark the dependency with `@Autowired(required = false)` and check for null before use.

```java
@Autowired(required = false)
public void setFraudService(FraudDetectionService fraudService) {
    this.fraudService = fraudService;
}
```

**Field Injection** uses `@Autowired` directly on fields. This is the discouraged approach because dependencies are hidden — they don't appear in the constructor signature, making it unclear what the class actually needs. It also makes unit testing harder because you need reflection (via `@InjectMocks`) or Spring context to inject dependencies. Field injection also allows dependencies to be mutable (non-final fields), which violates immutability principles.

In a banking context where auditability and testability are paramount, constructor injection ensures that the dependency graph is explicit and the service cannot be instantiated in an incomplete state. This is critical when processing financial transactions where missing a dependency could cause silent failures.

**Key Points to Hit:**
- [ ] Constructor injection = explicit, immutable, testable (preferred)
- [ ] Setter injection = optional dependencies only
- [ ] Field injection = hidden dependencies, hard to test (avoid)
- [ ] Constructor injection doesn't need `@Autowired` annotation
- [ ] Too many constructor parameters may indicate SRP violation

**Follow-Up Questions:**
1. What happens when you have a circular dependency with constructor injection?
2. How does Lombok's `@RequiredArgsConstructor` work with Spring?
3. Can you mix constructor injection with setter injection in the same class?

**Source:** `backend-engineering/java/dependency-injection.md`

---

### Q3: What are Spring Boot profiles and how would you use them in a banking environment?

**Strong Answer:**

Spring Boot profiles allow you to define different bean configurations and property sets for different environments. They are activated using `spring.profiles.active` property or the `SPRING_PROFILES_ACTIVE` environment variable.

In a banking environment, you typically have multiple profiles: `dev`, `test`, `staging`, `prod`, and sometimes specialized ones like `compliance` or `audit`. Each profile has its own `application-{profile}.yml` file.

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:h2:mem:devdb
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true

# application-prod.yml
spring:
  datasource:
    url: ${DATABASE_URL}
    hikari:
      maximum-pool-size: 50
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
```

Beyond configuration files, you can use `@Profile` annotations on `@Bean` methods to load entirely different implementations:

```java
@Configuration
public class EnvironmentConfig {
    @Bean
    @Profile("dev")
    public PaymentGateway mockPaymentGateway() {
        return new MockPaymentGateway();
    }

    @Bean
    @Profile("prod")
    public PaymentGateway productionPaymentGateway() {
        return new ProductionPaymentGateway();
    }
}
```

In a regulated banking context, profiles are critical for compliance. The `prod` profile enforces strict validation (`ddl-auto: validate`), disables SQL logging, and connects to production databases. The `dev` profile allows schema auto-generation and uses in-memory databases. Profile-specific beans also let you swap fraud detection implementations — a lightweight mock in `dev` vs. an AI-powered service in `prod`.

**Key Points to Hit:**
- [ ] Profile-specific YAML files (`application-{profile}.yml`)
- [ ] `@Profile` annotation on bean methods
- [ ] Activation via `spring.profiles.active` or environment variable
- [ ] Banking context: different configs for dev vs. prod (schema validation, logging, connection pools)
- [ ] Environment variables override profile defaults for sensitive data

**Follow-Up Questions:**
1. How do you set the active profile in a Docker container?
2. Can you activate multiple profiles simultaneously?
3. What is the difference between `@Profile` and `@ConditionalOnProperty`?

**Source:** `backend-engineering/java/dependency-injection.md`, `backend-engineering/java/spring-boot.md`

---

### Q4: What is the Spring IoC container and how does it manage bean lifecycle?

**Strong Answer:**

The Spring IoC (Inversion of Control) container is the core of the Spring Framework. It is responsible for instantiating beans, resolving their dependencies, applying post-processors, and managing their lifecycle. There are two main implementations: `BeanFactory` (the basic interface) and `ApplicationContext` (the production-ready implementation with additional features like event publishing, internationalization, and AOP).

The bean lifecycle follows these stages:

1. **Instantiation**: Spring creates the bean instance using reflection (constructor).
2. **Populate Properties**: Dependencies are injected via constructor, setter, or field injection.
3. **Aware Interfaces**: If the bean implements `BeanNameAware`, `BeanFactoryAware`, or `ApplicationContextAware`, Spring calls the corresponding methods.
4. **BeanPostProcessor (Before)**: `postProcessBeforeInitialization()` is called on all registered `BeanPostProcessor` instances.
5. **Initialization**: `@PostConstruct` methods or `InitializingBean.afterPropertiesSet()` are invoked. Custom `init-method` from `@Bean(initMethod = "...")` also runs here.
6. **BeanPostProcessor (After)**: `postProcessAfterInitialization()` runs — this is where AOP proxies are created.
7. **Ready**: Bean is available for use in the application.
8. **Destruction**: On context shutdown, `@PreDestroy` or `DisposableBean.destroy()` is called.

In a banking payment service, this lifecycle is crucial. For example, a `ConnectionPool` bean might use `@PostConstruct` to initialize the pool and `@PreDestroy` to gracefully close all connections before shutdown. The `BeanPostProcessor` stage is where `@Transactional` proxies are wrapped around service beans, adding transaction management transparently.

Understanding this lifecycle is essential for debugging startup issues, implementing custom bean processing, and ensuring proper resource cleanup — especially important in banking systems where database connections and message queue listeners must be gracefully shut down during deployments.

**Key Points to Hit:**
- [ ] IoC container creates, wires, and manages bean lifecycle
- [ ] ApplicationContext vs. BeanFactory
- [ ] Lifecycle stages: instantiation, dependency injection, aware callbacks, post-processors, initialization, ready, destruction
- [ ] `@PostConstruct` and `@PreDestroy` annotations
- [ ] AOP proxies are created during BeanPostProcessor (after) stage

**Follow-Up Questions:**
1. What is the difference between `@PostConstruct` and `InitializingBean`?
2. How does Spring create proxies for `@Transactional` methods?
3. What happens to singleton beans during a graceful shutdown?

**Source:** `backend-engineering/java/dependency-injection.md`

---

### Q5: How do you structure a production-grade Spring Boot service for a banking payment system?

**Strong Answer:**

A production-grade Spring Boot service follows a layered architecture with clear separation of concerns:

```
com.bank.payment/
├── PaymentServiceApplication.java   # @SpringBootApplication entry point
├── config/                          # Configuration classes
│   ├── SecurityConfig.java          # OAuth2, JWT, method security
│   ├── DatabaseConfig.java          # DataSource, HikariCP, JPA
│   ├── KafkaConfig.java             # Event streaming setup
│   └── OpenApiConfig.java           # Swagger/OpenAPI documentation
├── controller/                      # REST endpoints
│   ├── PaymentController.java       # Payment CRUD operations
│   └── HealthController.java        # Liveness/readiness probes
├── service/                         # Business logic
│   ├── PaymentService.java          # Core payment processing
│   └── ComplianceService.java       # Audit logging, rule checks
├── repository/                      # Data access (JPA)
│   ├── PaymentRepository.java
│   └── AccountRepository.java
├── model/                           # JPA entities
│   ├── Payment.java
│   ├── Account.java
│   └── AuditLog.java
├── dto/                             # Request/Response objects
│   ├── PaymentRequest.java          # Input validation
│   └── PaymentResponse.java         # Output mapping
├── exception/                       # Exception handling
│   ├── GlobalExceptionHandler.java  # @RestControllerAdvice
│   └── InsufficientFundsException.java
└── listener/                        # Event listeners
    └── PaymentEventListener.java    # Async event processing
```

Key design principles for banking:
- **Controllers** are thin — they delegate to services and handle HTTP concerns only
- **Services** contain business logic and are annotated with `@Transactional`
- **Repositories** extend `JpaRepository` for standard CRUD with custom `@Query` methods
- **DTOs** use Java records with validation annotations (`@NotBlank`, `@DecimalMin`)
- **GlobalExceptionHandler** uses `@RestControllerAdvice` for consistent error responses with audit-friendly error codes
- **Configuration** is externalized via `application.yml` with environment variable overrides for secrets
- **Health checks** verify database connectivity and Kafka availability for Kubernetes readiness probes

This structure supports auditability (every operation produces an audit log), compliance (validation at the DTO layer), and maintainability (clear layer boundaries that auditors and new team members can navigate).

**Key Points to Hit:**
- [ ] Layered architecture: controller, service, repository, model, DTO, config
- [ ] Thin controllers, fat services
- [ ] Java records for DTOs with validation annotations
- [ ] Global exception handling with `@RestControllerAdvice`
- [ ] Health checks for liveness/readiness probes

**Follow-Up Questions:**
1. Where should transaction boundaries be defined?
2. How do you handle cross-cutting concerns like logging and metrics?
3. What is the role of the listener layer in event-driven architecture?

**Source:** `backend-engineering/java/spring-boot.md`

---

### Q6: How does the `@Transactional` annotation work, and what are the different propagation levels?

**Strong Answer:**

The `@Transactional` annotation in Spring is implemented using AOP proxies. When a method annotated with `@Transactional` is called, Spring wraps the call in a transactional interceptor that:

1. Opens a database transaction (or joins an existing one based on propagation)
2. Executes the method
3. Commits the transaction if successful, or rolls back on `RuntimeException` or `Error`
4. Releases the connection back to the pool

**Important caveat**: `@Transactional` only works when the method is called from **outside the bean** (through the proxy). Self-invocation (calling a `@Transactional` method from another method in the same class) bypasses the proxy and the transaction is not applied.

**Propagation levels** define how transactions interact with existing transactions:

- **REQUIRED** (default): Join existing transaction or create a new one. Most common for banking operations.
- **REQUIRES_NEW**: Suspend the current transaction and start a new one. Useful for audit logging — you want the audit record committed even if the outer transaction rolls back.
- **MANDATORY**: Must run within an existing transaction; throws exception if none exists.
- **NEVER**: Must NOT run within a transaction; throws exception if one exists.
- **NOT_SUPPORTED**: Execute non-transactionally, suspending any current transaction.
- **SUPPORTS**: Join if exists, otherwise execute non-transactionally.
- **NESTED**: Creates a savepoint within the existing transaction (JDBC-specific, not JPA).

In a banking payment service:

```java
@Service
@RequiredArgsConstructor
public class PaymentService {
    private final AuditService auditService;

    @Transactional  // REQUIRED — default
    public Payment processPayment(PaymentCommand cmd) {
        // Entire payment processing is one transaction
        // If any step fails, all changes roll back
        Payment payment = createPayment(cmd);
        updateBalances(cmd);
        return payment;
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logAuditEvent(String event) {
        // This runs in its own transaction
        // Audit log persists even if outer transaction rolls back
        auditRepo.save(new AuditLog(event));
    }
}
```

For read-only operations, always use `@Transactional(readOnly = true)` to allow the database to optimize the query and avoid unnecessary write locks:

```java
@Transactional(readOnly = true)
public Optional<Payment> getPayment(String id) {
    return paymentRepo.findById(id);
}
```

**Key Points to Hit:**
- [ ] AOP proxy-based implementation
- [ ] Self-invocation bypasses the proxy
- [ ] `REQUIRED` (default) vs. `REQUIRES_NEW` (independent transaction)
- [ ] `readOnly = true` for read operations
- [ ] Rollback on `RuntimeException`, not checked exceptions (unless configured)

**Follow-Up Questions:**
1. Why doesn't `@Transactional` work on private methods?
2. How do you configure rollback for checked exceptions?
3. What happens to the transaction when an `@Async` method is called?

**Source:** `backend-engineering/java/spring-boot.md`, `backend-engineering/java/README.md`

---

### Q7: What is the N+1 query problem in JPA/Hibernate, and how do you solve it?

**Strong Answer:**

The N+1 query problem occurs when you fetch a collection of parent entities and then access a lazy-loaded child collection for each parent. This results in 1 query to fetch the parents and N additional queries (one per parent) to fetch the children — hence N+1 queries total.

```java
@Entity
public class Account {
    @OneToMany(mappedBy = "account")
    private List<Transaction> transactions;  // LAZY by default
}

// BAD: Triggers N+1
List<Account> accounts = accountRepo.findAll();  // 1 query
for (Account acc : accounts) {
    acc.getTransactions().size();  // N queries (one per account)
}
```

For 1,000 accounts, this generates 1,001 database queries — a catastrophic performance issue in a banking system with millions of accounts.

**Solutions:**

1. **`@EntityGraph`** (preferred): Declaratively specify fetch paths on repository methods.

```java
public interface AccountRepository extends JpaRepository<Account, String> {
    @EntityGraph(attributePaths = {"transactions"})
    @Query("SELECT a FROM Account a WHERE a.customerId = :customerId")
    List<Account> findByCustomerIdWithTransactions(@Param("customerId") String id);
}
```

2. **`JOIN FETCH` in JPQL**: Explicitly fetch the association in the query.

```java
@Query("SELECT DISTINCT a FROM Account a LEFT JOIN FETCH a.transactions WHERE a.id = :id")
Optional<Account> findByIdWithTransactions(@Param("id") String id);
```

3. **Batch fetching**: Configure Hibernate to load lazy collections in batches.

```yaml
spring.jpa.properties.hibernate.default_batch_fetch_size=50
```

4. **Use DTO projections**: Instead of loading full entities, project only the needed fields.

```java
@Query("SELECT new com.bank.dto.AccountSummary(a.id, a.balance, SIZE(a.transactions)) FROM Account a")
List<AccountSummary> getAccountSummaries();
```

**Configuration to prevent N+1**: Set `spring.jpa.open-in-view: false` in `application.yml`. The default `true` keeps the EntityManager open during view rendering, which silently triggers lazy loading in the controller layer. In a banking context, this should always be disabled to make query boundaries explicit and predictable.

The common mistake noted in the academy is: "Not using `@EntityGraph` for eager loading" — this is one of the most frequent performance issues in enterprise Java applications.

**Key Points to Hit:**
- [ ] N+1 = 1 parent query + N child queries for lazy collections
- [ ] `@EntityGraph` as the preferred solution
- [ ] `JOIN FETCH` in JPQL for explicit fetching
- [ ] Batch fetching with `hibernate.default_batch_fetch_size`
- [ ] `open-in-view: false` to prevent silent lazy loading in controllers

**Follow-Up Questions:**
1. What is the difference between `JOIN` and `JOIN FETCH` in JPQL?
2. When would you use a DTO projection instead of entity fetching?
3. How does `@EntityGraph` differ from `fetch = FetchType.EAGER`?

**Source:** `backend-engineering/java/performance.md`, `backend-engineering/java/spring-boot.md`

---

### Q8: How do you configure and monitor connection pooling for a high-throughput payment service?

**Strong Answer:**

Spring Boot uses HikariCP as the default connection pool. For a high-throughput payment service at a bank, proper configuration is critical to prevent connection exhaustion under load.

**Configuration:**

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20        # Max connections to database
      minimum-idle: 5              # Minimum idle connections maintained
      idle-timeout: 300000         # 5 minutes — close idle connections
      max-lifetime: 1800000        # 30 minutes — max connection lifetime
      connection-timeout: 30000    # 30 seconds — fail fast if pool exhausted
```

Sizing the pool: The formula is `connections = ((core_count * 2) + effective_spindle_count)`. For a cloud database with 8 cores, start around 20 connections. Too large a pool causes contention at the database level; too small causes thread waiting.

**Monitoring in production:**

```java
@Scheduled(fixedRate = 60000)
public void logPoolStats() {
    HikariPoolMXBean pool = dataSource.getHikariPoolMXBean();
    log.info("DB Pool: active={}, idle={}, waiting={}",
        pool.getActiveConnections(),
        pool.getIdleConnections(),
        pool.getThreadsAwaitingConnection());
}
```

You should also expose Hikari metrics via Micrometer to Prometheus/Grafana for alerting. Key metrics to monitor:
- **Active connections**: If consistently near `maximum-pool-size`, the pool is undersized
- **Threads awaiting connection**: If > 0, requests are being blocked waiting for connections
- **Connection timeout errors**: If connections can't be acquired within `connection-timeout`

**Production patterns for banking:**
- Set `connection-timeout` to 30 seconds (fail fast rather than hanging requests)
- Set `max-lifetime` to 30 minutes (shorter than database-side timeout to avoid stale connections)
- Use environment variables for pool sizes to tune per environment
- Never set `maximum-pool-size` to `Integer.MAX_VALUE` — a common mistake that causes database connection exhaustion

In a distributed microservices architecture, each service has its own pool. A 50-service architecture with 20 connections per service means 1,000 concurrent connections to databases — plan accordingly.

**Key Points to Hit:**
- [ ] HikariCP is Spring Boot's default connection pool
- [ ] Key settings: `maximum-pool-size`, `minimum-idle`, `connection-timeout`, `max-lifetime`
- [ ] Pool sizing formula based on CPU cores
- [ ] Monitor active/idle/waiting connections via `HikariPoolMXBean`
- [ ] Expose metrics via Micrometer for production alerting

**Follow-Up Questions:**
1. How do you determine the optimal pool size for your workload?
2. What happens when the connection pool is exhausted?
3. How does connection pooling interact with `@Transactional`?

**Source:** `backend-engineering/java/spring-boot.md`, `backend-engineering/java/performance.md`

---

### Q9: How do you implement a circuit breaker for an external payment gateway using Resilience4j?

**Strong Answer:**

A circuit breaker prevents cascading failures when an external service (like a payment gateway) becomes unavailable. Resilience4j is the Spring Cloud Circuit Breaker implementation recommended by Spring Boot.

The circuit breaker has three states:
- **CLOSED**: Requests flow normally. Failures are counted.
- **OPEN**: Failure rate exceeds threshold. Requests fail fast without calling the service.
- **HALF-OPEN**: After a wait duration, a limited number of requests are allowed through to test if the service has recovered.

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

    public CompletableFuture<PaymentResult> gatewayFallback(
            PaymentRequest request, Throwable t) {
        log.warn("Payment gateway unavailable, queueing: {}", request);
        paymentQueue.enqueue(request);
        return CompletableFuture.completedFuture(
            new PaymentResult("QUEUED", "Payment queued for later processing"));
    }
}
```

**Configuration:**

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentGateway:
        failure-rate-threshold: 50          # Open circuit at 50% failure rate
        wait-duration-in-open-state: 30s    # Wait 30s before half-open
        sliding-window-size: 10             # Measure over last 10 calls
        minimum-number-of-calls: 5          # Minimum calls before measuring
        automatic-transition-from-open-to-half-open-enabled: true
  retry:
    instances:
      paymentGateway:
        max-attempts: 3
        wait-duration: 1s
        retry-exceptions:
          - org.springframework.web.client.ResourceAccessException
  timelimiter:
    instances:
      paymentGateway:
        timeout-duration: 10s
```

**Banking context**: In a payment processing system, you cannot simply return an error to the customer when the external gateway is down. The fallback method queues the payment for asynchronous retry, ensuring no financial transaction is lost. This is a regulatory requirement — banks must guarantee eventual processing of all valid payment requests. The circuit breaker also protects against wasting resources on calls that will inevitably fail, preserving the service's capacity for other operations.

**Key Points to Hit:**
- [ ] Three states: CLOSED, OPEN, HALF-OPEN
- [ ] `@CircuitBreaker` + `@Retry` + `@TimeLimiter` can be stacked
- [ ] Fallback method provides graceful degradation (queue for retry in banking)
- [ ] Configuration via `resilience4j.*.instances.*` in YAML
- [ ] Banking context: never lose a payment transaction, always queue for retry

**Follow-Up Questions:**
1. What is the difference between sliding window size and minimum number of calls?
2. How do you monitor circuit breaker state changes in production?
3. When should you use a bulkhead pattern alongside a circuit breaker?

**Source:** `backend-engineering/java/microservices.md`

---

### Q10: How do you implement service-to-service communication with OpenFeign in a microservices architecture?

**Strong Answer:**

OpenFeign provides a declarative HTTP client for Spring Cloud microservices. Instead of manually constructing `RestTemplate` or `WebClient` calls, you define a Java interface with annotations, and Spring generates the implementation at runtime.

```java
@FeignClient(
    name = "account-service",
    url = "${services.account-service.url}",
    configuration = AccountServiceClientConfig.class
)
public interface AccountServiceClient {
    @GetMapping("/v1/accounts/{accountId}")
    ResponseEntity<AccountResponse> getAccount(@PathVariable String accountId);

    @PostMapping("/v1/accounts/{accountId}/debit")
    ResponseEntity<Void> debit(@PathVariable String accountId,
                               @RequestBody DebitRequest request);
}

@Service
@RequiredArgsConstructor
public class PaymentService {
    private final AccountServiceClient accountClient;

    public Payment processPayment(PaymentCommand cmd) {
        AccountResponse account = accountClient.getAccount(cmd.fromAccount()).getBody();
        if (account.getBalance().compareTo(cmd.amount()) < 0) {
            throw new InsufficientFundsException(cmd.fromAccount());
        }
        accountClient.debit(cmd.fromAccount(), new DebitRequest(cmd.amount()));
        // ...
    }
}
```

**Key advantages over `RestTemplate`:**
- Declarative — the interface serves as documentation of the service contract
- Built-in integration with Spring Cloud LoadBalancer for client-side load balancing
- Automatic integration with Resilience4j circuit breakers via `fallback` attribute
- Distributed tracing headers are automatically propagated (trace IDs flow across services)

**Configuration best practices for banking:**

```yaml
services:
  account-service:
    url: http://account-service:8080  # Use service name in Kubernetes

feign:
  client:
    config:
      default:
        connect-timeout: 5000
        read-timeout: 30000
        logger-level: BASIC  # Never FULL in production (logs sensitive data)
```

**Important considerations:**
- Always combine Feign clients with circuit breakers — synchronous calls to downstream services are a failure point
- In Kubernetes, use the service name as the URL (Kubernetes DNS resolves it), not Eureka
- Set appropriate timeouts to prevent thread pool exhaustion when downstream services are slow
- Use DTOs (not JPA entities) as request/response types to maintain service boundaries

In a banking microservices architecture, Feign clients are the primary mechanism for synchronous communication between services like payment, account, compliance, and notification services.

**Key Points to Hit:**
- [ ] Declarative interface replaces manual HTTP client code
- [ ] Integrates with Spring Cloud LoadBalancer and circuit breakers
- [ ] Automatic distributed tracing header propagation
- [ ] Configure timeouts to prevent thread exhaustion
- [ ] In Kubernetes, use service DNS names, not Eureka

**Follow-Up Questions:**
1. How do you add authentication headers to Feign requests?
2. What happens when the downstream service is slow — how do you configure timeouts?
3. How do you test a Feign client in isolation?

**Source:** `backend-engineering/java/microservices.md`

---

### Q11: How do you test a Spring Boot service with real PostgreSQL using Testcontainers?

**Strong Answer:**

Testcontainers runs real infrastructure (PostgreSQL, Kafka, Redis) in Docker containers during tests, providing production-accurate integration tests. This is far superior to using H2 in-memory database, which has different SQL dialects, lacks PostgreSQL-specific features (JSONB, arrays, window functions), and masks production bugs.

```java
@SpringBootTest
@Testcontainers
class PaymentControllerIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("banking_test")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        @DynamicPropertySource
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void createPayment_ValidRequest_Returns201() {
        PaymentRequest request = new PaymentRequest(
            "ACC-001", "ACC-002", new BigDecimal("100.00"), "USD", "Test payment");

        ResponseEntity<PaymentResponse> response = restTemplate.postForEntity(
            "/v1/payments", request, PaymentResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody().getStatus()).isEqualTo("PENDING");
    }

    @Test
    @Sql("/test-data/payments.sql")
    void getPayment_ExistingPayment_Returns200() {
        ResponseEntity<PaymentResponse> response = restTemplate.getForEntity(
            "/v1/payments/PAY-001", PaymentResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
    }
}
```

**Key patterns:**
- `@Container` + `static` ensures the container starts once for the entire test class (not per test)
- `@DynamicPropertySource` overrides Spring Boot properties with container's dynamic port/credentials
- `@Sql` loads test data from SQL scripts for reproducible test state
- `@SpringBootTest` loads the full application context for end-to-end testing

**Kafka testing** follows the same pattern:

```java
@Container
static KafkaContainer kafka = new KafkaContainer(
    DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));

@DynamicPropertySource
static void configureKafka(DynamicPropertyRegistry registry) {
    registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
}
```

**Banking context**: In a regulated environment, you need confidence that your payment service works against the actual database engine used in production. H2 would not catch PostgreSQL-specific issues like constraint violation messages, JSON serialization behavior, or timezone handling. Testcontainers provides this confidence while keeping tests automated and reproducible.

**Common mistake to avoid**: The academy lists "Using H2 instead of PostgreSQL in tests" as a common mistake. The academy also warns against using `@SpringBootTest` for unit tests — reserve it for integration tests only. Unit tests should use `@Mock` and `@InjectMocks` with Mockito.

**Key Points to Hit:**
- [ ] `@Testcontainers` + `@Container` runs real Docker containers in tests
- [ ] `@DynamicPropertySource` overrides Spring properties with container config
- [ ] Testcontainers vs. H2: real database engine catches dialect-specific bugs
- [ ] Use `@SpringBootTest` for integration tests, `@Mock`/`@InjectMocks` for unit tests
- [ ] KafkaContainer for event-driven integration tests

**Follow-Up Questions:**
1. How do you share a Testcontainer across multiple test classes?
2. What is the performance cost of Testcontainers vs. H2?
3. How do you handle test data cleanup between tests?

**Source:** `backend-engineering/java/testing.md`, `backend-engineering/java/spring-boot.md`

---

### Q12: What is the saga pattern, and how do you implement it for a transfer spanning multiple microservices?

**Strong Answer:**

The saga pattern manages distributed transactions across multiple microservices without using two-phase commit (2PC), which is impractical in a microservices architecture. Instead of a single atomic transaction, a saga is a sequence of local transactions, each with a compensating action that undoes the work if a later step fails.

```java
@Service
@RequiredArgsConstructor
public class TransferSaga {
    private final AccountServiceClient accountClient;
    private final PaymentRepository paymentRepo;

    @Transactional
    public TransferResult executeTransfer(TransferCommand cmd) {
        List<Compensation> compensations = new ArrayList<>();

        try {
            // Step 1: Debit source account
            accountClient.debit(cmd.fromAccount(), new DebitRequest(cmd.amount()));
            compensations.add(() ->
                accountClient.credit(cmd.fromAccount(), new CreditRequest(cmd.amount())));

            // Step 2: Credit destination account
            accountClient.credit(cmd.toAccount(), new CreditRequest(cmd.amount()));
            compensations.add(() ->
                accountClient.debit(cmd.toAccount(), new DebitRequest(cmd.amount())));

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
                    // Critical: alert on-call for manual intervention
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

**Key principles:**
- Each step is a local transaction within its own service
- Compensations run in **reverse order** to undo completed steps
- If a compensation fails, alert on-call — manual intervention may be needed (this is a banking system — you cannot lose money)
- The saga should be idempotent — retries should not double-credit accounts

**Banking context**: A fund transfer from Account A to Account B may involve the account service (debit/credit), the payment service (record the transaction), and the notification service (send confirmation). If the payment service fails after the accounts are updated, the saga must reverse the debit and credit. In a regulated banking environment, failed compensations trigger alerts to operations teams for manual reconciliation — data integrity is non-negotiable.

**Alternative approaches:**
- **Event-driven saga**: Use Kafka events to coordinate steps asynchronously (better for eventual consistency)
- **Choreography**: Each service listens to events and triggers the next step (no central coordinator)
- **Orchestration**: A central saga coordinator directs the steps (easier to reason about, as shown above)

**Key Points to Hit:**
- [ ] Saga = sequence of local transactions with compensating actions
- [ ] Compensations run in reverse order on failure
- [ ] Orchestration (central coordinator) vs. Choreography (event-driven)
- [ ] Banking: failed compensations must alert on-call for manual intervention
- [ ] Idempotency is critical — retries must not double-process

**Follow-Up Questions:**
1. What are the trade-offs between choreography and orchestration?
2. How do you handle a compensation that itself fails?
3. How do you track saga state for debugging and audit purposes?

**Source:** `backend-engineering/java/microservices.md`

---

### Q13: How do you implement distributed tracing across multiple microservices?

**Strong Answer:**

Distributed tracing tracks a single request as it flows through multiple microservices, assigning a unique trace ID and span IDs to each hop. This is essential for debugging performance issues and understanding request flows in a banking microservices architecture.

Spring Boot uses **Micrometer Tracing** (which replaced Spring Cloud Sleuth) with exporters like Zipkin or Jaeger.

**Configuration:**

```yaml
management:
  tracing:
    sampling:
      probability: 1.0  # Sample all traces in dev; reduce to 0.1 in production
  zipkin:
    tracing:
      endpoint: ${ZIPKIN_URL:http://zipkin:9411}/api/v2/spans
```

**How it works:**
1. A request enters the API Gateway, which generates a trace ID and span ID
2. These IDs are propagated via HTTP headers (`X-B3-TraceId`, `X-B3-SpanId`, `X-B3-ParentSpanId`) to downstream services
3. Each service creates a new span (child of the parent span) and propagates the trace ID
4. Spans are asynchronously sent to Zipkin/Jaeger for aggregation and visualization

**Log correlation**: You can include trace IDs in log output for correlation:

```yaml
logging:
  pattern:
    console: "%d{HH:mm:ss.SSS} [%thread] [%X{traceId},%X{spanId}] %-5level %logger - %msg%n"
```

This means every log line across all services includes the trace ID, allowing you to grep logs for a specific transaction.

**Feign client integration**: Trace IDs are automatically propagated through OpenFeign clients — no additional code needed.

```java
@FeignClient(name = "account-service")
public interface AccountServiceClient {
    // Trace ID automatically propagated via Spring Cloud
    @GetMapping("/v1/accounts/{id}")
    AccountResponse getAccount(@PathVariable String id);
}
```

**Banking context**: When a payment fails, operations teams need to trace the request across the payment service, account service, compliance service, and notification service. Without distributed tracing, debugging requires manually correlating timestamps across multiple services' logs — a time-consuming process during production incidents. With tracing, a single trace ID reveals the complete path and latency of each hop.

**Sampling**: In production, set `probability: 0.1` (sample 10%) to avoid overwhelming the tracing backend. In dev/test, use `1.0` to trace everything.

**Key Points to Hit:**
- [ ] Micrometer Tracing replaces Spring Cloud Sleuth
- [ ] Trace ID propagated via HTTP headers across services
- [ ] Log pattern includes traceId/spanId for correlation
- [ ] Sampling rate: 1.0 in dev, lower (0.1) in production
- [ ] Feign clients auto-propagate trace IDs

**Follow-Up Questions:**
1. How do you trace asynchronous events (Kafka messages) across services?
2. What is the difference between a trace, a span, and a log?
3. How do you use trace data to identify performance bottlenecks?

**Source:** `backend-engineering/java/microservices.md`

---

### Q14: How do you configure JVM for a latency-sensitive payment API?

**Strong Answer:**

For a latency-sensitive payment API, the primary concern is minimizing GC pause times and ensuring consistent response times. A payment request should complete in under 200ms — GC pauses of even 100ms can cause SLA violations.

**Garbage Collector Selection:**

```bash
# ZGC (Java 15+) — ultra-low pause times (< 1ms), best for latency-sensitive APIs
java -XX:+UseZGC \
     -XX:ZCollectionInterval=5 \
     -jar app.jar

# G1GC (default since Java 9) — good balance, configurable pause target
java -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=100 \
     -XX:G1HeapRegionSize=4m \
     -jar app.jar
```

For payment APIs, **ZGC is preferred** if using Java 15+ because it guarantees sub-millisecond pauses. G1GC is acceptable with `MaxGCPauseMillis=100` for Java 11-14 deployments.

**Container-aware memory settings:**

```bash
java -XX:+UseContainerSupport \
     -XX:MaxRAMPercentage=75.0 \
     -jar app.jar
```

This allocates 75% of container memory to the heap, leaving 25% for metaspace, thread stacks, direct buffers, and GC overhead. For a 4GB container: 3GB heap, 1GB non-heap.

**Dockerfile configuration:**

```dockerfile
ENTRYPOINT ["java", \
    "-XX:+UseZGC", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-jar", "app.jar"]
```

**Monitoring:** Use Java Flight Recorder (JFR) for production profiling:

```bash
java -XX:StartFlightRecording=duration=60s,filename=recording.jfr -jar app.jar
jmc recording.jfr  # Analyze with JDK Mission Control
```

Key metrics to monitor:
- GC pause times (target: < 10ms for ZGC, < 100ms for G1GC)
- GC frequency (too frequent = heap too small)
- Allocation rate (high allocation = potential memory leak)
- Thread contention (lock contention causes latency spikes)

**Banking context**: Payment SLAs are contractually obligated with customers and partners. A GC pause during peak trading hours or payment processing windows (e.g., end-of-day batch processing) can cause cascading failures. The academy notes that "default GC causes unpredictable latency" — never ship a production payment service without explicit GC configuration.

**Key Points to Hit:**
- [ ] ZGC for sub-millisecond pauses (Java 15+), G1GC with `MaxGCPauseMillis` as alternative
- [ ] `-XX:MaxRAMPercentage=75.0` for container-aware memory allocation
- [ ] `-XX:+UseContainerSupport` for Kubernetes/Docker environments
- [ ] JFR + Mission Control for production profiling
- [ ] Default GC settings cause unpredictable latency — always configure explicitly

**Follow-Up Questions:**
1. When would you choose G1GC over ZGC?
2. How do you detect a memory leak in production?
3. What is the impact of increasing heap size on GC behavior?

**Source:** `backend-engineering/java/performance.md`, `backend-engineering/java/spring-boot.md`

---

### Q15: What is the difference between `@Mock` and `@MockBean`, and when should you use each?

**Strong Answer:**

The distinction is fundamental to writing fast, correct tests in Spring Boot.

**`@Mock`** (from Mockito) creates a standalone mock object. It is used with `@InjectMocks` in **unit tests** that do NOT load the Spring context:

```java
@ExtendWith(MockitoExtension.class)
class PaymentServiceTest {
    @Mock
    private PaymentRepository paymentRepo;

    @Mock
    private AccountRepository accountRepo;

    @InjectMocks
    private PaymentService paymentService;  // Manually constructed with mocks

    @Test
    void processPayment_Success() {
        when(accountRepo.findById("ACC-001")).thenReturn(Optional.of(mockAccount));
        // ...
    }
}
```

**`@MockBean`** (from Spring Boot Test) creates a mock bean and **replaces the corresponding bean in the Spring ApplicationContext**. It requires `@SpringBootTest` or `@WebMvcTest`, which loads the Spring context:

```java
@WebMvcTest(PaymentController.class)
class PaymentControllerWebTest {
    @MockBean
    private PaymentService paymentService;  // Replaces bean in Spring context

    @Autowired
    private MockMvc mockMvc;

    @Test
    void createPayment_ValidRequest_Returns201() throws Exception {
        when(paymentService.processPayment(any())).thenReturn(mockPayment);
        mockMvc.perform(post("/v1/payments")...)
            .andExpect(status().isCreated());
    }
}
```

**Key differences:**

| Aspect | `@Mock` | `@MockBean` |
|--------|---------|-------------|
| Context | No Spring context | Replaces bean in Spring context |
| Speed | Fast (milliseconds) | Slow (seconds — context loading) |
| Usage | Unit tests with `@InjectMocks` | Integration tests with `@SpringBootTest`/`@WebMvcTest` |
| Isolation | Test-level | ApplicationContext-level (shared across tests) |

**When to use each:**
- Use `@Mock` + `@InjectMocks` for **unit tests** of service-layer logic. These run in milliseconds and test business logic in isolation.
- Use `@MockBean` for **slice tests** (`@WebMvcTest`, `@DataJpaTest`) where you need the Spring context for a specific layer but want to mock out dependencies.

**Common mistakes** from the academy:
- "Using `@SpringBootTest` for unit tests: Slow startup, unnecessary context" — use `@Mock` instead
- "Over-mocking: Mocking the class under test" — never mock what you're testing
- "No assertion on mock interactions: Not verifying what was called" — always `verify()` important interactions

**Banking context**: In a payment service, unit tests with `@Mock` verify the business logic (balance checks, fee calculations) run in milliseconds during CI. Integration tests with `@MockBean` verify the HTTP layer (controller validation, error handling) runs with real Spring MVC machinery but mocked services.

**Key Points to Hit:**
- [ ] `@Mock` = Mockito standalone mock, no Spring context, used with `@InjectMocks`
- [ ] `@MockBean` = replaces bean in Spring ApplicationContext, used with `@SpringBootTest`
- [ ] Unit tests use `@Mock` (fast); slice tests use `@MockBean` (context loading)
- [ ] `@MockBean` is slower due to Spring context loading
- [ ] Never mock the class under test; don't use `@SpringBootTest` for unit tests

**Follow-Up Questions:**
1. How does `@MockBean` affect test execution order and caching?
2. What is `@SpyBean` and when would you use it?
3. How do you reset `@MockBean` state between test methods?

**Source:** `backend-engineering/java/testing.md`

---

### Q16: How do you implement optimistic locking in JPA to handle concurrent balance updates?

**Strong Answer:**

Optimistic locking is a concurrency control mechanism that detects conflicting updates without using database-level locks. It is the preferred approach for banking applications where concurrent balance updates are common and pessimistic locking (SELECT FOR UPDATE) would cause unacceptable contention.

**Implementation:**

```java
@Entity
public class Account {
    @Id
    private String id;

    private BigDecimal balance;

    @Version
    private Long version;  // Optimistic locking column
}
```

The `@Version` annotation tells JPA to include a version column. When you update an entity, JPA checks that the version in the database matches the version in the entity. If they differ (another transaction modified the row), JPA throws `OptimisticLockException`.

```java
@Service
@RequiredArgsConstructor
public class AccountService {
    private final AccountRepository accountRepo;

    @Transactional
    public Account debitAccount(String accountId, BigDecimal amount) {
        Account account = accountRepo.findById(accountId)
            .orElseThrow(() -> new AccountNotFoundException(accountId));

        if (account.getBalance().compareTo(amount) < 0) {
            throw new InsufficientFundsException(accountId, amount);
        }

        account.setBalance(account.getBalance().subtract(amount));
        // On save, Hibernate checks version:
        // UPDATE account SET balance = ?, version = version + 1
        // WHERE id = ? AND version = ?
        // If version doesn't match, OptimisticLockException is thrown
        return accountRepo.save(account);
    }
}
```

**Handling `OptimisticLockException` in a banking context:**

```java
@Retryable(
    value = OptimisticLockException.class,
    maxAttempts = 3,
    backoff = @Backoff(delay = 100)
)
@Transactional
public Account processConcurrentDebit(String accountId, BigDecimal amount) {
    return debitAccount(accountId, amount);
}
```

When a conflict is detected, you retry the operation (read fresh data, apply the update, attempt save again). The `@Retryable` annotation from Spring Retry handles this automatically with exponential backoff.

**Why optimistic over pessimistic:**
- Optimistic locking scales well under low-to-moderate contention (most banking operations don't hit the same account simultaneously)
- Pessimistic locking (`SELECT FOR UPDATE`) holds database locks, causing thread blocking and potential deadlocks
- Optimistic locking allows the database to handle concurrency naturally with row-level version checks

**Trade-off**: Under high contention (e.g., flash sale events), optimistic locking causes many retries, increasing latency. In these cases, consider partitioning accounts or using a queue-based approach.

**Key Points to Hit:**
- [ ] `@Version` annotation on a `Long` field enables optimistic locking
- [ ] JPA checks version on UPDATE; throws `OptimisticLockException` on mismatch
- [ ] Handle conflicts with retry (`@Retryable`) with backoff
- [ ] Preferred over pessimistic locking for scalability
- [ ] Under high contention, consider queueing or partitioning

**Follow-Up Questions:**
1. What happens if two transactions try to debit the same account simultaneously?
2. How does optimistic locking interact with `@Transactional` isolation levels?
3. Can you use a timestamp column instead of a version number?

**Source:** `backend-engineering/java/performance.md`, `backend-engineering/java/spring-boot.md`, `backend-engineering/java/README.md`

---

### Q17: How do you design an event-driven architecture for payment processing using Spring Cloud Stream and Kafka?

**Strong Answer:**

Event-driven architecture decouples services by communicating through events rather than synchronous HTTP calls. For payment processing, this enables asynchronous processing, replay capability, and natural audit trails.

**Event Publisher (Payment Service):**

```java
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
```

**Event Consumer (Notification Service):**

```java
@Component
@Slf4j
public class PaymentEventConsumer {
    @Bean
    public Consumer<PaymentCompletedEvent> handlePaymentCompleted() {
        return event -> {
            log.info("Payment completed: {}", event.getPaymentId());
            notificationService.sendConfirmation(event);
            analyticsService.recordTransaction(event);
        };
    }
}
```

**Configuration:**

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

**Architecture for banking payments:**

```
Payment Service → Kafka Topic (payments.completed) → Multiple Consumers
  │                                                         │
  ├──► Notification Service (send email/SMS)                │
  ├──► Compliance Service (run AML checks)                  │
  ├──► Analytics Service (update dashboards)                │
  └──► Ledger Service (update general ledger)               │
```

**Key design decisions:**

1. **Partitioning**: Use account ID as the partition key to ensure all events for the same account are processed in order
2. **Consumer groups**: Each service has its own consumer group, so they independently receive all events
3. **Idempotent consumers**: Each consumer must handle duplicate events (Kafka may redeliver)
4. **Dead letter topics**: Configure DLQ for events that fail processing after retries
5. **Event schema**: Use a versioned event structure to support backward-compatible schema evolution

**Banking context**: In a regulated environment, the event log serves as an audit trail. Every payment produces events that compliance, risk, and analytics teams consume independently. The event-driven approach means adding a new consumer (e.g., a fraud detection service) requires zero changes to the payment service — it just subscribes to the topic.

**vs. synchronous calls**: Synchronous service-to-service calls (Feign) are appropriate for real-time validation (checking account balance before debiting). Events are appropriate for downstream processing that doesn't need to block the payment response (notifications, analytics, compliance screening).

**Key Points to Hit:**
- [ ] `StreamBridge` for publishing, functional `Consumer` beans for consuming
- [ ] Partition key ensures ordering per account
- [ ] Consumer groups allow independent consumption by multiple services
- [ ] Idempotent consumers handle duplicate deliveries
- [ ] Dead letter topics for failed event processing
- [ ] Banking: event log serves as audit trail

**Follow-Up Questions:**
1. How do you handle event ordering when a consumer falls behind?
2. What happens when a consumer crashes mid-processing?
3. How do you version event schemas without breaking existing consumers?

**Source:** `backend-engineering/java/microservices.md`, `backend-engineering/java/spring-boot.md`

---

### Q18: How do you detect and fix memory leaks in a long-running Java service?

**Strong Answer:**

Memory leaks in Java occur when objects are no longer needed but remain reachable from GC roots, preventing garbage collection. In a long-running banking service, memory leaks cause gradual heap growth, increasing GC frequency, and eventually `OutOfMemoryError`.

**Common causes in enterprise Java:**

1. **Static collections growing unbounded**: A `static Map` caching data without eviction
2. **Unclosed resources**: Database connections, file streams, HTTP clients not closed
3. **ThreadLocal variables not cleared**: Thread pool reuse means ThreadLocal values persist
4. **Listeners/callbacks not unregistered**: Event listeners keep references to old objects
5. **Classloader leaks**: In application servers, old classloaders not garbage collected

**Detection approach:**

1. **Monitor heap usage** via Micrometer/Prometheus. A steadily growing heap that doesn't drop after full GC indicates a leak.

2. **Heap dump analysis** when OOM occurs:
```bash
# Configure JVM to dump heap on OOM
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/app/heapdump.hprof

# Analyze with Eclipse MAT or JDK Mission Control
jmap -dump:live,format=b,file=heap.bin <pid>
```

3. **Async Profiler** for allocation profiling:
```bash
./profiler.sh -d 30 -e alloc -f alloc.svg <pid>
```
This shows which code paths allocate the most memory.

4. **JFR recording** for production-safe profiling:
```bash
java -XX:StartFlightRecording=duration=300s,filename=recording.jfr -jar app.jar
```

**Fixing common leaks:**

```java
// BAD: Unbounded static cache
private static final Map<String, Account> cache = new HashMap<>();

// GOOD: Bounded cache with eviction
private final Cache<String, Account> cache = CacheBuilder.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(5, TimeUnit.MINUTES)
    .recordStats()
    .build();

// BAD: Unclosed resources
Connection conn = dataSource.getConnection();
PreparedStatement stmt = conn.prepareStatement(sql);
// If exception occurs, conn is never closed

// GOOD: try-with-resources
try (Connection conn = dataSource.getConnection();
     PreparedStatement stmt = conn.prepareStatement(sql)) {
    // ...
}

// BAD: ThreadLocal not cleared
threadLocal.set(someObject);

// GOOD: Clear in finally
try {
    threadLocal.set(someObject);
    // ...
} finally {
    threadLocal.remove();
}
```

**Banking context**: A payment processing service running 24/7 cannot afford memory leaks. A leak that adds 1MB/hour will exhaust a 4GB heap in ~170 days — likely discovered during a critical processing window. Regular heap analysis should be part of production monitoring, and heap dumps on OOM should be configured in all environments.

**Key Points to Hit:**
- [ ] Common causes: unbounded caches, unclosed resources, ThreadLocal leaks, orphaned listeners
- [ ] Detection: heap dumps, async profiler allocation profiling, JFR recording
- [ ] Prevention: bounded caches (Guava/Caffeine), try-with-resources, clear ThreadLocal in finally
- [ ] `-XX:+HeapDumpOnOutOfMemoryError` for automatic heap dump on crash
- [ ] Monitoring: track heap usage trend over time via Prometheus/Grafana

**Follow-Up Questions:**
1. How do you analyze a 4GB heap dump without running out of memory on your analysis machine?
2. What is the difference between a memory leak and high memory usage?
3. How does GraalVM native image affect memory leak detection?

**Source:** `backend-engineering/java/performance.md`

---

### Q19: How do you implement batch operations in Hibernate for bulk data processing?

**Strong Answer:**

Without batch configuration, Hibernate executes each INSERT/UPDATE as an individual SQL statement. For bulk operations (end-of-day settlement, regulatory reporting data loads, migration scripts), this is prohibitively slow.

**Configuration:**

```yaml
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
```

- `batch_size=50`: Hibernate groups 50 statements into a single JDBC batch
- `order_inserts=true`: Groups INSERTs by entity type for optimal batching
- `order_updates=true`: Groups UPDATEs by entity type

**Batch insert pattern:**

```java
@Transactional
public void bulkInsertPayments(List<Payment> payments) {
    for (int i = 0; i < payments.size(); i++) {
        entityManager.persist(payments.get(i));
        if (i % 50 == 0) {
            entityManager.flush();    // Execute batch
            entityManager.clear();    // Clear persistence context (prevent OOM)
        }
    }
    // Flush remaining
    entityManager.flush();
    entityManager.clear();
}
```

**Critical**: The `entityManager.clear()` call is essential. Without it, the persistence context (first-level cache) grows with every persisted entity, eventually causing `OutOfMemoryError`. The `flush()` sends the batched SQL to the database, and `clear()` detaches all entities from the persistence context.

**Batch updates** follow the same pattern:

```java
@Transactional
public void bulkUpdateAccountStatuses(List<Account> accounts) {
    for (int i = 0; i < accounts.size(); i++) {
        entityManager.merge(accounts.get(i));
        if (i % 50 == 0) {
            entityManager.flush();
            entityManager.clear();
        }
    }
    entityManager.flush();
    entityManager.clear();
}
```

**Alternative: JPQL bulk operations**

For operations that don't need entity lifecycle callbacks, use JPQL bulk updates/deletes:

```java
@Modifying
@Query("UPDATE Account a SET a.status = :status WHERE a.lastActivityDate < :cutoff")
int deactivateInactiveAccounts(@Param("status") AccountStatus status,
                               @Param("cutoff") LocalDate cutoff);
```

This executes as a single SQL UPDATE statement — much faster than loading, modifying, and saving each entity. However, it bypasses JPA lifecycle callbacks and doesn't update the persistence context.

**Banking context**: End-of-day batch processing in banks involves settling thousands of transactions, updating account balances, generating regulatory reports, and archiving old records. These operations run during maintenance windows and must complete within strict time limits. Proper batching reduces a 2-hour process to 5 minutes. The `order_inserts` and `order_updates` settings are particularly important when processing mixed entity types, as they allow the JDBC driver to batch efficiently.

**Key Points to Hit:**
- [ ] `hibernate.jdbc.batch_size=50` enables JDBC batching
- [ ] `order_inserts` and `order_updates` group statements by entity type
- [ ] `flush()` + `clear()` every N iterations to prevent OOM
- [ ] JPQL bulk operations for cases not needing entity lifecycle
- [ ] Banking: end-of-day batch processing must complete within maintenance windows

**Follow-Up Questions:**
1. What is the optimal batch size and how do you determine it?
2. When would you use JDBC batch vs. JPQL bulk operations?
3. How do you handle failures mid-batch?

**Source:** `backend-engineering/java/performance.md`

---

### Q20: How would you design a reactive, streaming-capable GenAI backend service using Spring WebFlux?

**Strong Answer:**

For a GenAI backend (like an AI-powered financial analysis service), responses can take 30+ seconds. Blocking a thread per request (traditional Spring MVC) wastes resources. Spring WebFlux provides non-blocking, reactive processing that streams responses as they are generated.

**WebFlux Controller for streaming LLM responses:**

```java
@RestController
@RequestMapping("/v1/ai")
@RequiredArgsConstructor
public class AiAnalysisController {
    private final AiAnalysisService aiService;

    @PostMapping(value = "/analyze", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<String>> analyze(
            @RequestBody AiAnalysisRequest request) {
        return aiService.streamAnalysis(request)
            .map(chunk -> ServerSentEvent.<String>builder()
                .event("analysis")
                .data(chunk)
                .build())
            .onErrorResume(error -> Flux.just(
                ServerSentEvent.<String>builder()
                    .event("error")
                    .data("{\"error\": \"" + error.getMessage() + "\"}")
                    .build()));
    }

    @GetMapping(value = "/health", produces = MediaType.APPLICATION_JSON_VALUE)
    public Mono<Map<String, String>> health() {
        return Mono.just(Map.of("status", "healthy"));
    }
}
```

**Reactive Service with LLM client:**

```java
@Service
@RequiredArgsConstructor
public class AiAnalysisService {
    private final WebClient webClient;  // Reactive HTTP client

    public Flux<String> streamAnalysis(AiAnalysisRequest request) {
        return webClient.post()
            .uri("/v1/chat/completions")
            .header("Authorization", "Bearer ${AI_API_KEY}")
            .bodyValue(Map.of(
                "model", "gpt-4",
                "messages", List.of(Map.of(
                    "role", "user",
                    "content", request.getPrompt()
                )),
                "stream", true
            ))
            .retrieve()
            .bodyToFlux(String.class)
            .map(this::parseSSEChunk)
            .timeout(Duration.ofSeconds(60))
            .doOnError(error -> log.error("AI analysis failed", error));
    }

    private String parseSSEChunk(String sseData) {
        // Parse Server-Sent Events chunk from LLM response
        // ...
    }
}
```

**Key differences from Spring MVC:**
- `Flux<T>` = stream of 0..N items (reactive stream)
- `Mono<T>` = stream of 0..1 item (single async result)
- `WebClient` replaces `RestTemplate` (non-blocking HTTP client)
- No thread-per-request model — uses event loop (Netty) with few threads
- Must use non-blocking drivers for databases (R2DBC instead of JDBC)

**Banking GenAI context**: A financial analysis service might stream real-time insights as the LLM generates them — the user sees partial results immediately rather than waiting for the full response. This is particularly valuable for long-running tasks like regulatory document analysis, portfolio risk assessment, or fraud pattern detection.

**Architecture considerations:**
- Use WebFlux for the AI-facing layer (streaming responses), but Spring MVC for traditional REST APIs (payment processing) — they can coexist in the same application
- R2DBC for reactive database access (though JPA is blocking — consider hybrid approach)
- Circuit breakers are still needed (resilience4j-reactor for WebFlux)
- Kafka integration works with WebFlux via Spring Cloud Stream's reactive bindings
- Thread pool configuration differs — Netty event loops replace Tomcat thread pools

**Trade-offs:**
- WebFlux has a steeper learning curve (functional reactive programming)
- Debugging is harder (stack traces span reactive chains)
- Most third-party libraries are blocking — you may need `publishOn(Schedulers.boundedElastic())` to wrap blocking calls
- In a banking context with existing Spring MVC infrastructure, a hybrid approach is pragmatic: WebFlux for the AI streaming layer, Spring MVC for everything else

**Key Points to Hit:**
- [ ] `Flux<T>` for streaming, `Mono<T>` for single async results
- [ ] `WebClient` replaces `RestTemplate` for reactive HTTP
- [ ] Server-Sent Events (SSE) for streaming LLM responses to clients
- [ ] Netty event loop model replaces thread-per-request
- [ ] Hybrid architecture: WebFlux for AI layer, Spring MVC for traditional APIs
- [ ] R2DBC for reactive database access (vs. blocking JDBC)

**Follow-Up Questions:**
1. How do you handle backpressure when the LLM streams faster than the client can consume?
2. How do you integrate blocking JPA repositories with WebFlux?
3. What is the difference between `subscribeOn()` and `publishOn()`?

**Source:** `backend-engineering/java/microservices.md`, `backend-engineering/java/spring-boot.md` (adapted for GenAI context)
