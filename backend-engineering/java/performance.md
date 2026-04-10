# Java Performance Engineering

## JVM Tuning

### Garbage Collection

```bash
# G1GC (default since Java 9) - good balance of throughput and latency
java -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=100 \
     -XX:G1HeapRegionSize=4m \
     -jar app.jar

# ZGC (Java 15+) - ultra-low pause times (< 1ms)
java -XX:+UseZGC \
     -XX:ZCollectionInterval=5 \
     -jar app.jar

# Shenandoah GC (OpenJDK) - low pause times
java -XX:+UseShenandoahGC \
     -jar app.jar

# For latency-sensitive services (payment APIs)
# Use ZGC or G1GC with low pause target

# For throughput-oriented services (batch processing)
# Use G1GC with larger pause target
```

### Memory Configuration

```bash
# Container-aware memory settings (Java 10+)
java -XX:+UseContainerSupport \
     -XX:MaxRAMPercentage=75.0 \
     -jar app.jar

# 75% of container memory goes to heap
# Remaining 25% for: metaspace, thread stacks, direct buffers, GC

# For 4GB container:
# Heap: 3GB (75%)
# Non-heap: 1GB

# Traditional fixed heap (avoid in containers)
java -Xms2g -Xmx2g -jar app.jar
```

## Profiling

### Async Profiler

```bash
# CPU profiling
./profiler.sh -d 30 -f cpu.svg <pid>

# Memory allocation profiling
./profiler.sh -d 30 -e alloc -f alloc.svg <pid>

# Lock contention
./profiler.sh -d 30 -e lock -f lock.svg <pid>
```

### JFR (Java Flight Recorder)

```bash
# Start recording
java -XX:StartFlightRecording=duration=60s,filename=recording.jfr \
     -jar app.jar

# Analyze with JDK Mission Control
jmc recording.jfr

# Or convert to readable format
jfr print --events JDK.ObjectAllocationInNewTLAB recording.jfr
```

## Database Performance

### Hibernate Optimization

```java
// N+1 Query Problem
@Entity
public class Account {
    @OneToMany(mappedBy = "account")
    private List<Transaction> transactions;  // Lazy by default
}

// BAD: Triggers N+1 queries
List<Account> accounts = accountRepo.findAll();
for (Account acc : accounts) {
    acc.getTransactions().size();  // Triggers query per account
}

// GOOD: Use @EntityGraph
public interface AccountRepository extends JpaRepository<Account, String> {
    @EntityGraph(attributePaths = {"transactions"})
    @Query("SELECT a FROM Account a WHERE a.customerId = :customerId")
    List<Account> findByCustomerIdWithTransactions(@Param("customerId") String id);
}

// Or use JOIN FETCH
@Query("SELECT DISTINCT a FROM Account a LEFT JOIN FETCH a.transactions WHERE a.id = :id")
Optional<Account> findByIdWithTransactions(@Param("id") String id);
```

### Batch Operations

```java
// application.yml
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true

// Batch insert
@Transactional
public void bulkInsertPayments(List<Payment> payments) {
    for (int i = 0; i < payments.size(); i++) {
        entityManager.persist(payments.get(i));
        if (i % 50 == 0) {
            entityManager.flush();
            entityManager.clear();
        }
    }
}
```

## Connection Pool Monitoring

```java
@Configuration
public class DatabaseMonitoring {
    @Bean
    public HikariDataSource dataSource(HikariConfig config) {
        HikariDataSource ds = new HikariDataSource(config);

        // Monitor pool stats
        MBeanServer mBeanServer = ManagementFactory.getPlatformMBeanServer();
        try {
            mBeanServer.registerMBean(ds.getHikariPoolMXBean(),
                new ObjectName("com.zaxxer.hikari:type=Pool"));
        } catch (Exception e) {
            log.warn("Failed to register Hikari MBean", e);
        }

        return ds;
    }
}

// Log pool stats periodically
@Scheduled(fixedRate = 60000)
public void logPoolStats() {
    HikariPoolMXBean pool = dataSource.getHikariPoolMXBean();
    log.info("DB Pool: active={}, idle={}, waiting={}",
        pool.getActiveConnections(),
        pool.getIdleConnections(),
        pool.getThreadsAwaitingConnection());
}
```

## HTTP Performance

### RestTemplate Optimization

```java
@Configuration
public class HttpClientConfig {
    @Bean
    public RestTemplate restTemplate() {
        // Connection pooling
        PoolingHttpClientConnectionManager connectionManager =
            new PoolingHttpClientConnectionManager();
        connectionManager.setMaxTotal(100);
        connectionManager.setDefaultMaxPerRoute(20);

        CloseableHttpClient httpClient = HttpClients.custom()
            .setConnectionManager(connectionManager)
            .setKeepAliveStrategy((response, context) -> 60_000)
            .build();

        HttpComponentsClientHttpRequestFactory factory =
            new HttpComponentsClientHttpRequestFactory(httpClient);
        factory.setConnectTimeout(5000);
        factory.setReadTimeout(30000);

        return new RestTemplate(factory);
    }
}
```

## Memory Leak Detection

```java
// Common Java memory leaks:
// 1. Static collections growing unbounded
// 2. Unclosed resources (connections, streams)
// 3. Thread-local variables not cleared
// 4. Listener/callback not unregistered
// 5. Classloader leaks (in application servers)

// Prevention:
// - Use try-with-resources for AutoCloseable
// - Limit collection sizes (use bounded caches)
// - Clear ThreadLocal in finally blocks
// - Use WeakReference for caches

// Example: Bounded cache
private final Cache<String, Account> cache = CacheBuilder.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(5, TimeUnit.MINUTES)
    .recordStats()
    .build();
```

## Common Mistakes

1. **Default GC settings**: GC pauses cause latency spikes
2. **N+1 queries in JPA**: Not using EntityGraph or JOIN FETCH
3. **No connection pool monitoring**: Pool exhaustion in production
4. **Not using batch operations**: Individual inserts for bulk data
5. **Memory leaks**: Static collections, unclosed resources
6. **Ignoring container memory**: Not using `-XX:MaxRAMPercentage`

## Interview Questions

1. How do you configure JVM for a latency-sensitive payment API?
2. Design a strategy to detect and fix N+1 queries in JPA.
3. How do you profile a Java service with high memory usage?
4. What is the difference between G1GC and ZGC?
5. How do you monitor connection pool health in production?

## Cross-References

- `./spring-boot.md` - Spring Boot performance configuration
- `./testing.md` - Benchmarking with JMH
- `./microservices.md` - Performance in distributed systems
- `../performance/README.md` - Overall performance engineering
