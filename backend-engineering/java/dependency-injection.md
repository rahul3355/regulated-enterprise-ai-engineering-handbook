# Dependency Injection in Java

## Why DI

Dependency Injection decouples components from their implementations. In banking services, DI enables testing with mock repositories, swapping payment gateways, and managing transaction boundaries. Spring's DI container is the foundation of all Spring Boot applications.

## Constructor Injection (Recommended)

```java
@Service
@RequiredArgsConstructor  // Lombok generates constructor
@Slf4j
public class PaymentService {
    private final PaymentRepository paymentRepo;
    private final AccountRepository accountRepo;
    private final NotificationService notificationService;
    private final ApplicationEventPublisher eventPublisher;

    public Payment processPayment(PaymentCommand command) {
        // Use injected dependencies
        Account from = accountRepo.findById(command.fromAccount())
            .orElseThrow(() -> new AccountNotFoundException(command.fromAccount()));

        Payment payment = paymentRepo.save(Payment.from(command));
        notificationService.sendPaymentConfirmation(payment);
        eventPublisher.publishEvent(new PaymentCreatedEvent(payment));

        return payment;
    }
}
```

## Field Injection (Avoid)

```java
// BAD: Field injection (harder to test, hidden dependencies)
@Service
public class PaymentService {
    @Autowired
    private PaymentRepository paymentRepo;

    @Autowired
    private AccountRepository accountRepo;

    // Dependencies not visible in constructor
    // Cannot create instance without Spring container
    // Harder to unit test
}
```

## Setter Injection (Optional Dependencies)

```java
@Component
public class PaymentProcessor {
    private FraudDetectionService fraudService;

    @Autowired(required = false)  // Optional dependency
    public void setFraudService(FraudDetectionService fraudService) {
        this.fraudService = fraudService;
    }

    public void process(Payment payment) {
        if (fraudService != null) {
            fraudService.check(payment);
        }
        // Continue processing
    }
}
```

## Bean Configuration

```java
@Configuration
public class ServiceConfig {

    @Bean
    @Primary  // Default implementation when multiple exist
    public PaymentGateway primaryPaymentGateway(
            @Value("${payment.gateway.url}") String url,
            @Value("${payment.gateway.api-key}") String apiKey) {
        return new StripePaymentGateway(url, apiKey);
    }

    @Bean("backupPaymentGateway")
    public PaymentGateway backupPaymentGateway(
            @Value("${payment.gateway.backup.url}") String url) {
        return new BackupPaymentGateway(url);
    }

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplateBuilder()
            .setConnectTimeout(Duration.ofSeconds(5))
            .setReadTimeout(Duration.ofSeconds(30))
            .build();
    }
}
```

## Component Scanning

```java
@SpringBootApplication(scanBasePackages = "com.bank.payment")
// Equivalent to:
// @ComponentScan("com.bank.payment")
// @EnableAutoConfiguration
// @SpringBootConfiguration

// Stereotype annotations:
@Component       // Generic Spring bean
@Service         // Business logic layer
@Repository      // Data access layer (adds exception translation)
@Controller      // Web controller
@RestController  // REST controller (combines @Controller + @ResponseBody)
```

## Conditional Beans

```java
@Configuration
public class EnvironmentConfig {

    @Bean
    @ConditionalOnProperty(name = "features.fraud-detection", havingValue = "true")
    public FraudDetectionService fraudDetectionService() {
        return new AIFraudDetectionService();
    }

    @Bean
    @ConditionalOnMissingBean(FraudDetectionService.class)
    public FraudDetectionService defaultFraudDetectionService() {
        return new NoOpFraudDetectionService();
    }

    @Bean
    @ConditionalOnClass(name = "com.stripe.Stripe")
    public PaymentGateway stripeGateway() {
        return new StripeGateway();
    }

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

## Qualifier for Multiple Implementations

```java
public interface PaymentGateway {
    PaymentResult process(Payment payment);
}

@Service
@Qualifier("stripe")
public class StripeGateway implements PaymentGateway {
    public PaymentResult process(Payment payment) {
        // Stripe implementation
    }
}

@Service
@Qualifier("adyen")
public class AdyenGateway implements PaymentGateway {
    public PaymentResult process(Payment payment) {
        // Adyen implementation
    }
}

@Service
@RequiredArgsConstructor
public class PaymentService {
    private final PaymentGateway paymentGateway;

    // Or use @Qualifier at injection point
    public PaymentService(
            @Qualifier("stripe") PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
}
```

## Factory Bean

```java
@Configuration
public class DatabaseConfig {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.hikari")
    public HikariConfig hikariConfig() {
        return new HikariConfig();
    }

    @Bean
    public DataSource dataSource(HikariConfig config) {
        return new HikariDataSource(config);
    }
}
```

## Common Mistakes

1. **Field injection**: Hidden dependencies, hard to test
2. **Circular dependencies**: Service A depends on Service B depends on Service A
3. **Too many constructor parameters**: Service has 10+ dependencies (violates SRP)
4. **Not using interfaces**: Tied to concrete implementation
5. **`@Autowired` on everything**: Constructor injection doesn't need `@Autowired`
6. **No conditional beans**: Loading unnecessary beans in certain environments

## Interview Questions

1. What is the difference between constructor, setter, and field injection?
2. How do you handle multiple implementations of the same interface?
3. Design a configuration that loads different beans for dev vs prod.
4. What happens when there is a circular dependency?
5. How does `@Primary` differ from `@Qualifier`?

## Cross-References

- `./spring-boot.md` - Spring Boot service structure
- `./testing.md` - Testing with injected mocks
- `./microservices.md` - DI in microservices
- `../microservices/README.md` - Service dependencies
