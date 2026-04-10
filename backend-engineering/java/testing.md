# Java Testing Patterns

## JUnit 5

### Basic Tests

```java
@ExtendWith(MockitoExtension.class)
class PaymentServiceTest {
    @Mock
    private PaymentRepository paymentRepo;

    @Mock
    private AccountRepository accountRepo;

    @Mock
    private NotificationService notificationService;

    @InjectMocks
    private PaymentService paymentService;

    @Test
    @DisplayName("Successful payment processing")
    void processPayment_Success() {
        // Arrange
        String fromAccount = "ACC-001";
        String toAccount = "ACC-002";
        BigDecimal amount = new BigDecimal("100.00");

        Account fromAcc = new Account(fromAccount, new BigDecimal("1000.00"));
        Account toAcc = new Account(toAccount, new BigDecimal("500.00"));

        when(accountRepo.findById(fromAccount)).thenReturn(Optional.of(fromAcc));
        when(accountRepo.findById(toAccount)).thenReturn(Optional.of(toAcc));
        when(paymentRepo.save(any())).thenAnswer(i -> i.getArgument(0));

        PaymentCommand command = new PaymentCommand(fromAccount, toAccount, amount, "USD");

        // Act
        Payment result = paymentService.processPayment(command);

        // Assert
        assertNotNull(result);
        assertEquals(PaymentStatus.COMPLETED, result.getStatus());
        assertEquals(amount, result.getAmount());

        // Verify interactions
        verify(accountRepo).findById(fromAccount);
        verify(accountRepo).findById(toAccount);
        verify(paymentRepo).save(any());
        verify(notificationService).sendPaymentConfirmation(any());
    }

    @Test
    @DisplayName("Insufficient funds throws exception")
    void processPayment_InsufficientFunds() {
        // Arrange
        Account fromAcc = new Account("ACC-001", new BigDecimal("50.00"));
        when(accountRepo.findById("ACC-001")).thenReturn(Optional.of(fromAcc));

        PaymentCommand command = new PaymentCommand("ACC-001", "ACC-002",
            new BigDecimal("100.00"), "USD");

        // Act & Assert
        assertThrows(InsufficientFundsException.class, () ->
            paymentService.processPayment(command));
    }
}
```

### Parameterized Tests

```java
@ParameterizedTest
@DisplayName("Payment amount validation")
@ValueSource(strings = {"0.00", "-1.00", "-0.01"})
void processPayment_InvalidAmount_ThrowsError(String amountStr) {
    BigDecimal amount = new BigDecimal(amountStr);
    PaymentCommand command = new PaymentCommand("ACC-001", "ACC-002", amount, "USD");

    assertThrows(InvalidPaymentException.class, () ->
        paymentService.processPayment(command));
}

@ParameterizedTest
@CsvSource({
    "USD, true",
    "EUR, true",
    "GBP, true",
    "XYZ, false",
    ", false",
})
void validateCurrency_SupportedCurrency_ReturnsTrue(String currency, boolean expected) {
    boolean result = paymentService.isSupportedCurrency(currency);
    assertEquals(expected, result);
}
```

## Mockito

### Mocking External Services

```java
@ExtendWith(MockitoExtension.class)
class PaymentGatewayIntegrationTest {
    @Mock
    private RestTemplate restTemplate;

    @InjectMocks
    private ExternalPaymentGateway gateway;

    @Test
    void initiatePayment_Success() {
        // Arrange
        PaymentRequest request = new PaymentRequest("ACC-001", "100.00", "USD");

        PaymentResponse mockResponse = new PaymentResponse("EXT-PAY-001", "accepted");
        when(restTemplate.postForEntity(anyString(), any(), eq(PaymentResponse.class)))
            .thenReturn(ResponseEntity.ok(mockResponse));

        // Act
        PaymentResponse result = gateway.initiatePayment(request);

        // Assert
        assertEquals("EXT-PAY-001", result.getPaymentId());
        assertEquals("accepted", result.getStatus());
    }

    @Test
    void initiatePayment_Timeout_ThrowsException() {
        when(restTemplate.postForEntity(anyString(), any(), eq(PaymentResponse.class)))
            .thenThrow(new ResourceAccessException("Connection timed out"));

        assertThrows(PaymentGatewayTimeoutException.class, () ->
            gateway.initiatePayment(new PaymentRequest("ACC-001", "100.00", "USD")));
    }
}
```

## Test Containers

### PostgreSQL Integration Tests

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
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private PaymentRepository paymentRepo;

    @Test
    @Sql("/test-data/payments.sql")
    void getPayment_ExistingPayment_Returns200() {
        ResponseEntity<PaymentResponse> response = restTemplate.getForEntity(
            "/v1/payments/PAY-001", PaymentResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().getId()).isEqualTo("PAY-001");
    }

    @Test
    void createPayment_ValidRequest_Returns201() {
        PaymentRequest request = new PaymentRequest(
            "ACC-001", "ACC-002", new BigDecimal("100.00"), "USD", "Test payment"
        );

        ResponseEntity<PaymentResponse> response = restTemplate.postForEntity(
            "/v1/payments", request, PaymentResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody().getStatus()).isEqualTo("PENDING");
    }
}
```

### Kafka Integration Tests

```java
@SpringBootTest
@Testcontainers
class PaymentEventListenerIntegrationTest {
    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));

    @DynamicPropertySource
    static void configureKafka(DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    @Autowired
    private PaymentEventConsumer consumer;

    @Test
    void consumePaymentEvent_ProcessesSuccessfully() {
        // Send event
        PaymentCompletedEvent event = new PaymentCompletedEvent("PAY-001");
        kafkaTemplate.send("payments.completed", event).join();

        // Wait for consumer to process
        await().atMost(10, TimeUnit.SECONDS)
            .untilAsserted(() -> {
                assertThat(consumer.getProcessedEvents()).hasSize(1);
                assertThat(consumer.getProcessedEvents().get(0).getPaymentId())
                    .isEqualTo("PAY-001");
            });
    }
}
```

## WebMvc Testing

```java
@WebMvcTest(PaymentController.class)
class PaymentControllerWebTest {
    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private PaymentService paymentService;

    @Test
    void createPayment_ValidRequest_Returns201() throws Exception {
        Payment payment = new Payment("PAY-001", new BigDecimal("100.00"), "USD");
        when(paymentService.processPayment(any())).thenReturn(payment);

        mockMvc.perform(post("/v1/payments")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                        "fromAccount": "ACC-001",
                        "toAccount": "ACC-002",
                        "amount": "100.00",
                        "currency": "USD"
                    }
                    """))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value("PAY-001"))
            .andExpect(jsonPath("$.amount").value("100.00"));
    }

    @Test
    void createPayment_InvalidRequest_Returns400() throws Exception {
        mockMvc.perform(post("/v1/payments")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                        "fromAccount": "",
                        "amount": "-100"
                    }
                    """))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.code").value("VALIDATION_ERROR"));
    }
}
```

## Common Mistakes

1. **Testing implementation details**: Tests break on refactoring
2. **No test isolation**: Tests sharing database state
3. **Over-mocking**: Mocking the class under test
4. **Using `@SpringBootTest` for unit tests**: Slow startup, unnecessary context
5. **Not using test containers**: H2 != PostgreSQL, different SQL dialects
6. **No assertion on mock interactions**: Not verifying what was called

## Interview Questions

1. How do you test a Spring Boot controller that calls an external payment gateway?
2. Design integration tests for a payment service with real PostgreSQL.
3. What is the difference between `@MockBean` and `@Mock`?
4. How do you test a Kafka consumer in isolation?
5. What are the benefits of test containers over in-memory databases?

## Cross-References

- `./spring-boot.md` - Spring Boot testing configuration
- `./dependency-injection.md` - Testing with injected mocks
- `../backend-testing/README.md` - Overall testing strategy
- `../api-design/README.md` - Testing API contracts
