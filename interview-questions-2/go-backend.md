# Go Backend Interview Questions — 20 Questions with Full Answers

## Overview
| Detail | Value |
|--------|-------|
| Topic | Go Backend (Concurrency, gRPC, Performance, Service Structure, Testing) |
| Questions | 20 (5 Must-Know, 10 Medium, 5 Advanced) |
| Source Files | backend-engineering/go/ (7 files: README, concurrency, error-handling, grpc, performance, service-structure, testing) |
| Citi Relevance | Go is one of Citi's four backend languages. Excellent for high-throughput API gateways, gRPC microservices, and concurrent processing of LLM requests. |

---

## Difficulty Legend
- 🔵 **Must-Know** — Foundational. Every candidate should know these.
- 🟡 **Medium** — Core competency. What you'll actually be tested on.
- 🔴 **Advanced** — Differentiator. Shows principal-engineer depth.

---

### Q1: 🔵 How does Go's concurrency model differ from traditional OS threads, and why does it matter for a high-throughput payment gateway?

**Strong Answer:**

Go's concurrency model is built around **goroutines** — lightweight threads managed by the Go runtime scheduler, not the OS. A goroutine starts with a **2 KB stack** compared to 1-2 MB for OS threads. The Go runtime multiplexes thousands of goroutines onto a small number of OS threads using an M:N scheduler, enabling massive concurrency with minimal overhead.

In a high-throughput payment gateway processing 10,000 concurrent requests, this model is transformative. If each request spawned a traditional OS thread, you would need 10-20 GB of memory just for thread stacks. With goroutines, the same 10,000 concurrent requests consume roughly 20-40 MB for stacks. Additionally, Go has **no GIL (Global Interpreter Lock)**, so goroutines execute in true parallelism across CPU cores.

```go
// Processing 10,000 payments concurrently
func processPayments(payments []Payment) ([]Result, error) {
    results := make(chan Result, len(payments))
    var wg sync.WaitGroup

    for _, p := range payments {
        wg.Add(1)
        go func(payment Payment) {
            defer wg.Done()
            result := processSingle(payment)
            results <- result
        }(p)
    }

    wg.Wait()
    close(results)

    var allResults []Result
    for r := range results {
        allResults = append(allResults, r)
    }
    return allResults, nil
}
```

The trade-off is that goroutines are so cheap that developers can accidentally spawn unbounded numbers of them. In production banking services, you must constrain concurrency with **worker pools** or **semaphores** to prevent resource exhaustion on downstream systems like databases.

**Key Points to Hit:**
- [ ] Goroutines start at 2 KB stack vs 1-2 MB for OS threads
- [ ] Go runtime M:N scheduler multiplexes goroutines onto OS threads
- [ ] No GIL — true parallelism across CPU cores
- [ ] Memory efficiency enables thousands of concurrent operations
- [ ] Must still constrain concurrency to protect downstream resources

**Follow-Up Questions:**
1. How would you limit the number of concurrent goroutines processing payments?
2. What happens to a goroutine when it blocks on I/O?
3. How do you detect and prevent goroutine leaks?

**Source:** `backend-engineering/go/README.md`, `backend-engineering/go/concurrency.md`

---

### Q2: 🔵 What is the difference between buffered and unbuffered channels in Go, and when would you use each?

**Strong Answer:**

An **unbuffered channel** (`make(chan int)`) provides synchronous communication — the sender blocks until a receiver is ready, and the receiver blocks until a sender arrives. This creates a rendezvous point that guarantees synchronization between goroutines. It is ideal for signaling and coordination where you need to know the receiver has consumed the value.

A **buffered channel** (`make(chan int, 100)`) provides asynchronous communication — the sender blocks only when the buffer is full, and the receiver blocks only when the buffer is empty. The buffer acts as a queue, decoupling the sender and receiver in time. This is ideal for producer-consumer patterns, worker pools, and rate limiting.

```go
// Unbuffered: synchronous — sender waits for receiver
unbuffered := make(chan int)

// Buffered: asynchronous — holds up to 100 items before blocking
buffered := make(chan Result, 100)

// Worker pool with buffered channels
func processPayments(ctx context.Context, payments []Payment) ([]Result, error) {
    results := make(chan Result, len(payments))  // Buffered: no blocking on send
    errCh := make(chan error, 1)
    var wg sync.WaitGroup

    for _, p := range payments {
        wg.Add(1)
        go func(payment Payment) {
            defer wg.Done()
            result, err := processSingle(ctx, payment)
            if err != nil {
                select {
                case errCh <- err:  // Non-blocking: buffer size 1
                default:            // Already an error recorded
                }
                return
            }
            results <- result  // Won't block because channel is buffered
        }(p)
    }

    go func() {
        wg.Wait()
        close(results)
    }()

    var allResults []Result
    for result := range results {
        allResults = append(allResults, result)
    }

    select {
    case err := <-errCh:
        return allResults, err
    default:
        return allResults, nil
    }
}
```

In banking contexts, buffered channels are common in worker pools where you want to queue up tasks without blocking producers. Unbuffered channels are better for shutdown signals or when you need guaranteed hand-off (e.g., confirming a payment event was consumed before proceeding).

**Key Points to Hit:**
- [ ] Unbuffered channels block sender until receiver is ready (synchronous)
- [ ] Buffered channels block sender only when buffer is full (asynchronous)
- [ ] Buffered channels prevent producer blocking in worker pool patterns
- [ ] Unbuffered channels provide synchronization guarantees
- [ ] The `select` with `default` pattern creates non-blocking sends on buffered channels

**Follow-Up Questions:**
1. What happens if you send to a closed channel?
2. How do you safely close a channel when multiple goroutines may send?
3. Can you read from a nil channel? What happens?

**Source:** `backend-engineering/go/concurrency.md`

---

### Q3: 🔵 How do sentinel errors and error wrapping work together to provide context in Go?

**Strong Answer:**

**Sentinel errors** are predefined error values using `errors.New()` that represent specific, recognizable conditions. They serve as a contract between layers — a repository can return `model.ErrNotFound`, and a handler can check for it to return the appropriate HTTP status code. However, returning a raw sentinel error loses context about where and why the error occurred.

**Error wrapping** with `fmt.Errorf("context: %w", err)` adds contextual information while preserving the ability to check against the original sentinel error using `errors.Is()`. The `%w` verb creates a chain of wrapped errors that `errors.Is()` can traverse.

```go
// Sentinel errors defined once
var (
    ErrNotFound          = errors.New("resource not found")
    ErrInsufficientFunds = errors.New("insufficient funds")
    ErrAccountClosed     = errors.New("account closed")
)

// Repository wraps sentinel with context
func (r *PaymentRepository) GetByID(ctx context.Context, id string) (*Payment, error) {
    row := r.db.QueryRowContext(ctx, "SELECT * FROM payments WHERE id = $1", id)
    var p Payment
    if err := row.Scan(&p.ID, &p.Amount, &p.Status); err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, fmt.Errorf("getting payment by id %s: %w", id, ErrNotFound)
        }
        return nil, fmt.Errorf("scanning payment row: %w", err)
    }
    return &p, nil
}

// Handler checks the wrapped error
func (h *PaymentHandler) CreatePayment(w http.ResponseWriter, r *http.Request) {
    result, err := h.service.CreatePayment(r.Context(), cmd)
    if err != nil {
        switch {
        case errors.Is(err, model.ErrInsufficientFunds):
            respondError(w, http.StatusUnprocessableEntity, "INSUFFICIENT_FUNDS", err.Error())
        case errors.Is(err, model.ErrAccountClosed):
            respondError(w, http.StatusForbidden, "ACCOUNT_CLOSED", err.Error())
        case errors.Is(err, model.ErrNotFound):
            respondError(w, http.StatusNotFound, "NOT_FOUND", err.Error())
        default:
            h.logger.Error("Unexpected error", zap.Error(err))
            respondError(w, http.StatusInternalServerError, "INTERNAL_ERROR", "Internal server error")
        }
        return
    }
    respondJSON(w, http.StatusCreated, result)
}
```

In a banking environment, this pattern is critical. The sentinel errors define the API contract (client-facing error codes), while wrapping provides the audit trail needed for debugging in production. The full error chain — "transferring $7500 from account ACC-001: insufficient funds" — gives operations teams exactly what they need for incident response.

**Key Points to Hit:**
- [ ] Sentinel errors are predefined values for recognizable conditions
- [ ] `fmt.Errorf("context: %w", err)` wraps while preserving the original
- [ ] `errors.Is()` traverses the entire error chain to find sentinel matches
- [ ] Each layer adds context without losing the ability to check base errors
- [ ] In banking, sentinel errors define the client-facing contract; wrapping provides audit context

**Follow-Up Questions:**
1. When would you use `errors.As()` instead of `errors.Is()`?
2. What happens if you use `%v` instead of `%w` in fmt.Errorf?
3. How do you design custom error types with unwrap support?

**Source:** `backend-engineering/go/error-handling.md`

---

### Q4: 🔵 Describe the standard production service layout for a Go banking microservice and explain the purpose of each directory.

**Strong Answer:**

A production Go banking service follows a **clean architecture** pattern that separates concerns into distinct layers. The layout is standardized across the industry and enforced by Go's visibility rules.

```
banking-payment-service/
├── cmd/server/main.go          # Entry point: wiring, config, server startup
├── internal/                   # PRIVATE code — cannot be imported by other modules
│   ├── config/config.go        # Environment variable loading and validation
│   ├── handler/                # HTTP/gRPC handlers (parse requests, delegate to services)
│   ├── service/                # Business logic (payment processing, fraud detection)
│   ├── repository/             # Data access (SQL queries, Redis operations)
│   ├── model/                  # Domain models (Payment, Account, Money types)
│   ├── middleware/             # HTTP middleware (auth, logging, recovery, CORS)
│   └── server/                 # HTTP server configuration (timeouts, TLS)
├── pkg/                        # PUBLIC code — reusable across modules
│   └── validator/              # Shared validators (IBAN, SWIFT)
├── api/                        # API definitions (protobuf, OpenAPI specs)
├── deployments/
│   └── Dockerfile              # Multi-stage build for production images
├── testutil/                   # Test helpers and fixtures
├── go.mod / go.sum             # Dependency management
└── Makefile                    # Build automation
```

The critical distinction is `internal/` vs `pkg/`. Go enforces that packages under `internal/` **cannot be imported by other modules** — this is a compile-time guarantee that internal implementation details are not accidentally exposed. Packages under `pkg/` are public and reusable, like validation utilities shared across services.

```go
// cmd/server/main.go — dependency injection at the composition root
func main() {
    logger, _ := zap.NewProduction()
    cfg, _ := config.Load()

    db, _ := repository.NewDatabase(cfg.DatabaseURL)
    defer db.Close()

    rdb, _ := repository.NewRedis(cfg.RedisURL)

    paymentRepo := repository.NewPaymentRepository(db)
    accountRepo := repository.NewAccountRepository(db)
    paymentSvc := service.NewPaymentService(paymentRepo, accountRepo, logger)

    paymentHandler := handler.NewPaymentHandler(paymentSvc, logger)

    mux := http.NewServeMux()
    mux.Handle("POST /v1/payments", middleware.Chain(
        middleware.RequestID(),
        middleware.Logging(logger),
        middleware.Recovery(logger),
    ).Then(paymentHandler.CreatePayment))

    srv := server.New(mux, server.Options{
        Addr:         fmt.Sprintf(":%d", cfg.Port),
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 30 * time.Second,
        IdleTimeout:  120 * time.Second,
    })

    // Graceful shutdown
    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()
    go func() { srv.ListenAndServe() }()
    <-ctx.Done()
    srv.Shutdown(ctx)
}
```

In banking, this structure supports audit requirements: each layer has a single responsibility, making it easy to trace a payment from HTTP request through business logic to database commit.

**Key Points to Hit:**
- [ ] `internal/` is private — Go compiler prevents external imports
- [ ] `pkg/` is public — reusable across modules
- [ ] Handlers parse requests and delegate; services contain business logic; repositories handle data access
- [ ] `cmd/` is the composition root where dependency injection happens
- [ ] Graceful shutdown via `signal.NotifyContext` and `server.Shutdown`

**Follow-Up Questions:**
1. Why not put everything in `main.go` for a small service?
2. How do you handle configuration that differs between dev, staging, and prod?
3. What is the role of the `internal/middleware/` layer?

**Source:** `backend-engineering/go/service-structure.md`, `backend-engineering/go/README.md`

---

### Q5: 🔵 What is the `context.Context` in Go and why is it essential for every banking API?

**Strong Answer:**

`context.Context` is a built-in Go type that carries **deadlines, cancellation signals, and request-scoped values** across API boundaries and goroutines. It is the standard mechanism for controlling the lifecycle of operations in Go.

In banking APIs, context is essential for three reasons:

1. **Timeout enforcement**: Every database query, HTTP call, and gRPC request should accept a context with a timeout to prevent requests from hanging indefinitely and consuming resources.
2. **Graceful shutdown**: When the server receives a shutdown signal, the context is cancelled, which propagates to all in-flight operations, allowing them to clean up.
3. **Request-scoped values**: Context carries metadata like request IDs, user authentication claims, and trace IDs through the entire call chain.

```go
func ProcessPayment(ctx context.Context, payment Payment) error {
    // Check if context already cancelled
    select {
    case <-ctx.Done():
        return ctx.Err()
    default:
    }

    // Create child context with timeout
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel() // Always defer cancel to prevent context leak

    // Propagate context to database query
    row := db.QueryRowContext(ctx,
        "SELECT balance FROM accounts WHERE id = $1",
        payment.FromAccount,
    )

    // Propagate context to HTTP call to external gateway
    req, _ := http.NewRequestWithContext(ctx, "POST", gatewayURL, body)
    resp, err := client.Do(req)

    // Propagate context to goroutine for async notification
    go func() {
        ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
        defer cancel()
        sendNotification(ctx, payment)
    }()

    return nil
}
```

The critical rule is: **every long-running operation must accept and respect context**. In a regulated banking environment, this ensures that no request can run indefinitely, and the audit trail (via request IDs in context) connects every log entry to a specific client request.

**Key Points to Hit:**
- [ ] Context carries deadlines, cancellation, and request-scoped values
- [ ] `context.WithTimeout` creates a child context with a deadline
- [ ] Always `defer cancel()` to prevent context leaks
- [ ] Context must propagate through all layers: handler → service → repository
- [ ] In banking, context carries request IDs for distributed tracing/audit

**Follow-Up Questions:**
1. What is the difference between `context.WithCancel` and `context.WithTimeout`?
2. Should you ever use `context.Background()` in a request handler?
3. How do you store values in context and when is it appropriate?

**Source:** `backend-engineering/go/concurrency.md`, `backend-engineering/go/service-structure.md`

---

### Q6: 🟡 Design a worker pool pattern for processing 10,000 payments with a maximum of 50 concurrent workers.

**Strong Answer:**

A worker pool limits concurrency to protect downstream resources (databases, external APIs) while maximizing throughput. The pattern uses a buffered job channel feeding a fixed number of goroutines, with proper shutdown and error handling.

```go
type PaymentWorker struct {
    workerCount int
    jobQueue    chan Payment
    resultQueue chan PaymentResult
    ctx         context.Context
    cancel      context.CancelFunc
}

func NewPaymentWorker(workerCount int) *PaymentWorker {
    ctx, cancel := context.WithCancel(context.Background())
    return &PaymentWorker{
        workerCount: workerCount,
        jobQueue:    make(chan Payment, 1000),   // Buffer: queue up to 1000 pending jobs
        resultQueue: make(chan PaymentResult, 1000),
        ctx:         ctx,
        cancel:      cancel,
    }
}

func (w *PaymentWorker) Start() {
    for i := 0; i < w.workerCount; i++ {
        go w.worker(i)
    }
}

func (w *PaymentWorker) worker(id int) {
    for {
        select {
        case <-w.ctx.Done():
            log.Printf("Worker %d shutting down", id)
            return

        case payment, ok := <-w.jobQueue:
            if !ok {
                return  // Job queue closed
            }
            result := processPayment(payment)
            w.resultQueue <- result
        }
    }
}

func (w *PaymentWorker) Submit(payment Payment) error {
    select {
    case w.jobQueue <- payment:
        return nil
    case <-time.After(5 * time.Second):
        return fmt.Errorf("job queue full — payment rejected")
    }
}

func (w *PaymentWorker) Stop() {
    w.cancel()
    close(w.jobQueue)
}
```

**Usage in a banking context:**

```go
func ProcessBatchPayments(ctx context.Context, payments []Payment) ([]PaymentResult, error) {
    worker := NewPaymentWorker(50)
    worker.Start()

    // Submit all payments
    for _, p := range payments {
        if err := worker.Submit(p); err != nil {
            return nil, fmt.Errorf("submitting payment: %w", err)
        }
    }

    // Collect results
    var results []PaymentResult
    for i := 0; i < len(payments); i++ {
        select {
        case result := <-worker.resultQueue:
            results = append(results, result)
        case <-ctx.Done():
            worker.Stop()
            return results, ctx.Err()
        }
    }

    worker.Stop()
    return results, nil
}
```

Key design decisions: the job channel buffer (1000) provides backpressure — if the queue fills, `Submit` times out after 5 seconds, returning an error to the caller rather than blocking indefinitely. The context-based shutdown ensures all workers exit cleanly. In a Citi payment processing system, this pattern prevents database connection pool exhaustion by capping concurrency at 50, regardless of how many payments arrive.

**Key Points to Hit:**
- [ ] Fixed number of goroutines (50 workers) consume from a shared buffered channel
- [ ] `Submit` uses `select` with timeout to implement backpressure
- [ ] Context-based cancellation enables graceful shutdown
- [ ] Worker checks `ctx.Done()` and channel close (`ok` pattern) for clean exit
- [ ] Buffer size on job queue determines how much backlog is tolerated before rejecting

**Follow-Up Questions:**
1. How would you add retry logic for failed payments?
2. What happens if a worker panics during processing?
3. How do you track progress and report metrics from the worker pool?

**Source:** `backend-engineering/go/concurrency.md`

---

### Q7: 🟡 Explain the fan-out/fan-in pattern for concurrent processing and provide a banking-relevant example.

**Strong Answer:**

**Fan-out** means distributing work across multiple goroutines to process tasks in parallel. **Fan-in** means collecting results from those goroutines back into a single channel or data structure. Combined, they form a pattern for parallel processing with result aggregation.

In a banking context, consider fetching account balances for 100 accounts. Sequentially, each query takes 10 ms, so 100 accounts would take 1 second. With fan-out/fan-in using 20 goroutines, the total time drops to ~50 ms (the longest single query).

```go
func fetchAccountBalances(ctx context.Context, accountIDs []string) (map[string]decimal.Decimal, error) {
    var (
        mu      sync.Mutex
        wg      sync.WaitGroup
        results = make(map[string]decimal.Decimal)
        errCh   = make(chan error, len(accountIDs))
    )

    // Fan-out: launch a goroutine per account
    for _, accountID := range accountIDs {
        wg.Add(1)
        go func(id string) {
            defer wg.Done()

            balance, err := db.GetBalance(ctx, id)
            if err != nil {
                errCh <- fmt.Errorf("account %s: %w", id, err)
                return
            }

            mu.Lock()
            results[id] = balance
            mu.Unlock()
        }(accountID)
    }

    // Wait for all goroutines to complete
    wg.Wait()
    close(errCh)

    // Fan-in: collect errors
    var errs []error
    for err := range errCh {
        errs = append(errs, err)
    }
    if len(errs) > 0 {
        return results, fmt.Errorf("fetching balances: %w", errors.Join(errs...))
    }

    return results, nil
}
```

The `sync.Mutex` protects the shared `results` map from concurrent writes — without it, Go's race detector would flag a data race. The error channel collects failures, and `errors.Join()` combines multiple errors into a single wrapped error.

**Trade-offs**: Fan-out maximizes throughput but increases resource consumption (database connections, memory). In production, you would limit fan-out with a semaphore (buffered channel) to prevent overwhelming the database:

```go
sem := make(chan struct{}, 20) // Max 20 concurrent DB queries
for _, accountID := range accountIDs {
    wg.Add(1)
    go func(id string) {
        defer wg.Done()
        sem <- struct{}{}        // Acquire
        defer func() { <-sem }() // Release
        // ... fetch balance
    }(accountID)
}
```

**Key Points to Hit:**
- [ ] Fan-out: split work across multiple goroutines for parallel execution
- [ ] Fan-in: collect results and errors from all goroutines
- [ ] `sync.Mutex` protects shared state (the results map)
- [ ] `errors.Join()` combines multiple errors into one
- [ ] Use semaphores (buffered channels) to limit concurrency in production

**Follow-Up Questions:**
1. How do you handle partial failures when some accounts succeed and others fail?
2. What is the difference between this pattern and a worker pool?
3. How would you add a timeout to the entire fan-out operation?

**Source:** `backend-engineering/go/concurrency.md`, `backend-engineering/go/README.md`

---

### Q8: 🟡 What are the trade-offs between `sync.Mutex` and `sync.RWMutex`, and when should you use each in a banking service?

**Strong Answer:**

`sync.Mutex` provides exclusive access — only one goroutine can hold the lock at a time. `sync.RWMutex` allows **multiple concurrent readers OR one exclusive writer**. The choice depends on your read/write ratio and the cost of contention.

In banking services, **read-heavy caches** (account balances, currency exchange rates) are ideal for `RWMutex`:

```go
type AccountBalanceCache struct {
    mu       sync.RWMutex
    balances map[string]decimal.Decimal
}

func (c *AccountBalanceCache) Get(accountID string) (decimal.Decimal, bool) {
    c.mu.RLock()         // Multiple readers allowed — no blocking between readers
    defer c.mu.RUnlock()
    balance, ok := c.balances[accountID]
    return balance, ok
}

func (c *AccountBalanceCache) Set(accountID string, balance decimal.Decimal) {
    c.mu.Lock()          // Exclusive write — blocks all readers and writers
    defer c.mu.Unlock()
    c.balances[accountID] = balance
}
```

Use `sync.Mutex` (not `RWMutex`) when:
- Write frequency is comparable to reads (no benefit from reader optimization)
- The critical section is very short (lock overhead dominates)
- You need write starvation protection (RWMutex can starve writers under heavy read load)

Use `sync.RWMutex` when:
- Reads vastly outnumber writes (typical for caches, configuration)
- Read operations are expensive enough that reader parallelism matters
- You can tolerate occasional write starvation

**Important rule**: Never copy a struct containing a mutex. Pass by pointer:

```go
// WRONG — copying the mutex
func processCache(c AccountBalanceCache) { ... }

// CORRECT — pointer preserves mutex state
func processCache(c *AccountBalanceCache) { ... }
```

**Key Points to Hit:**
- [ ] `Mutex`: one goroutine at a time (exclusive)
- [ ] `RWMutex`: multiple readers OR one writer (shared read, exclusive write)
- [ ] RWMutex is optimal for read-heavy caches like exchange rate tables
- [ ] Never copy a struct containing a mutex — pass by pointer
- [ ] RWMutex can starve writers under heavy read load

**Follow-Up Questions:**
1. What is mutex starvation and how does it manifest?
2. When would you use `sync.Map` instead of a mutex-protected map?
3. How do you detect lock contention in production?

**Source:** `backend-engineering/go/concurrency.md`

---

### Q9: 🟡 How do you implement and test a rate limiter in Go for an API gateway?

**Strong Answer:**

A rate limiter using the **token bucket** algorithm limits the number of requests a client can make within a time window. The implementation uses a buffered channel as the token store, refilled at regular intervals by a background goroutine.

```go
type RateLimiter struct {
    tokens   chan struct{}
    interval time.Duration
}

func NewRateLimiter(maxTokens int, interval time.Duration) *RateLimiter {
    rl := &RateLimiter{
        tokens:   make(chan struct{}, maxTokens),
        interval: interval,
    }
    go func() {
        ticker := time.NewTicker(interval)
        for range ticker.C {
            select {
            case rl.tokens <- struct{}{}:
            default: // Bucket full — discard excess token
            }
        }
    }()
    return rl
}

func (rl *RateLimiter) Allow() bool {
    select {
    case <-rl.tokens:
        return true
    default:
        return false
    }
}
```

As HTTP middleware:

```go
limiter := NewRateLimiter(100, time.Second) // 100 requests per second

func rateLimitMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if !limiter.Allow() {
            http.Error(w, "rate limited", http.StatusTooManyRequests)
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

**Testing** uses table-driven tests with controlled timing:

```go
func TestRateLimiter(t *testing.T) {
    tests := []struct {
        name      string
        maxTokens int
        requests  int
        wantAllow int
    }{
        {"within limit", 5, 3, 3},
        {"at limit", 5, 5, 5},
        {"over limit", 5, 10, 5},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            rl := NewRateLimiter(tt.maxTokens, time.Hour) // Long interval so tokens don't refill
            allowed := 0
            for i := 0; i < tt.requests; i++ {
                if rl.Allow() {
                    allowed++
                }
            }
            if allowed != tt.wantAllow {
                t.Errorf("allowed %d, want %d", allowed, tt.wantAllow)
            }
        })
    }
}
```

In a Citi API gateway, rate limiting protects backend services from traffic spikes, prevents abuse, and ensures fair resource allocation across clients. Production implementations would use Redis-backed distributed rate limiters for multi-instance deployments.

**Key Points to Hit:**
- [ ] Token bucket: buffered channel holds tokens, ticker refills them
- [ ] `Allow()` uses non-blocking select — returns immediately if no token available
- [ ] Middleware rejects with 429 when rate limit exceeded
- [ ] Testing uses a long refill interval to prevent token regeneration during test
- [ ] Production needs distributed rate limiting (Redis) for multi-instance gateways

**Follow-Up Questions:**
1. How would you implement per-client rate limiting instead of global?
2. What is the sliding window algorithm and how does it compare to token bucket?
3. How do you handle rate limiter state when the service restarts?

**Source:** `backend-engineering/go/concurrency.md`

---

### Q10: 🟡 How do you handle validation errors spanning multiple fields in Go?

**Strong Answer:**

Single-field validation can return sentinel errors, but **multi-field validation** requires a custom error type that aggregates all validation failures. This allows the client to see all errors at once rather than fixing them one at a time.

```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error on field '%s': %s", e.Field, e.Message)
}

type ValidationErrors []*ValidationError

func (ve ValidationErrors) Error() string {
    if len(ve) == 0 {
        return ""
    }
    messages := make([]string, len(ve))
    for i, e := range ve {
        messages[i] = e.Error()
    }
    return strings.Join(messages, "; ")
}

func ValidatePayment(req CreatePaymentRequest) error {
    var errors ValidationErrors

    if req.Amount.LessThanOrEqual(decimal.Zero) {
        errors = append(errors, &ValidationError{
            Field:   "amount",
            Message: "must be greater than zero",
        })
    }
    if !IsValidCurrency(req.Currency) {
        errors = append(errors, &ValidationError{
            Field:   "currency",
            Message: fmt.Sprintf("unsupported currency: %s", req.Currency),
        })
    }
    if req.FromAccount == req.ToAccount {
        errors = append(errors, &ValidationError{
            Field:   "to_account",
            Message: "cannot pay to self",
        })
    }

    if len(errors) > 0 {
        return errors
    }
    return nil
}
```

In the handler, `ValidationErrors` can be checked with `errors.As()`:

```go
err := ValidatePayment(req)
if err != nil {
    var validationErrs model.ValidationErrors
    if errors.As(err, &validationErrs) {
        respondError(w, http.StatusBadRequest, "VALIDATION_ERROR", err.Error())
        return
    }
    // Some other error
    respondError(w, http.StatusInternalServerError, "INTERNAL_ERROR", "Internal error")
}
```

This pattern is essential in banking: regulatory requirements often mandate comprehensive input validation, and returning all errors at once improves the developer experience for clients integrating with the API.

**Key Points to Hit:**
- [ ] Custom `ValidationErrors` type implements the `error` interface
- [ ] Aggregates all field-level failures rather than failing on the first one
- [ ] `errors.As()` extracts the typed error from the error chain
- [ ] Each `ValidationError` carries field name and message for client feedback
- [ ] Banking regulations require comprehensive validation — returning all errors aids compliance

**Follow-Up Questions:**
1. How would you add JSON serialization to `ValidationErrors`?
2. When should validation happen in the handler vs the service layer?
3. How do you test validation with all combinations of invalid inputs?

**Source:** `backend-engineering/go/error-handling.md`

---

### Q11: 🟡 What are the four types of gRPC streaming and when would you use each in a banking microservice?

**Strong Answer:**

gRPC supports four RPC patterns, each suited to different communication needs:

**1. Unary RPC** — Single request, single response. The most common pattern, equivalent to a REST call.
```protobuf
rpc GetAccount(GetAccountRequest) returns (GetAccountResponse);
```
Use for: fetching account details, creating a single payment, health checks.

**2. Server Streaming** — Single request, server sends multiple responses.
```protobuf
rpc GetTransactions(GetTransactionsRequest) returns (stream Transaction);
```
Use for: paginated transaction history, streaming large result sets where each record is sent as soon as it is available rather than loading everything into memory.

**3. Client Streaming** — Client sends multiple requests, server returns single response.
```protobuf
rpc BulkCreateTransactions(stream CreateTransactionRequest) returns (BatchResult);
```
Use for: bulk uploads (e.g., end-of-day batch payments from a partner), where the client streams records and the server returns a summary of successes and failures.

**4. Bidirectional Streaming** — Both sides stream independently.
```protobuf
rpc MonitorAccount(MonitorAccountRequest) returns (stream AccountEvent);
```
Use for: real-time event feeds (balance changes, transaction notifications), chat-like interactions, or any scenario where both sides push data continuously.

**Banking example — bidirectional streaming for account monitoring:**

```go
func (s *AccountServer) MonitorAccount(
    req *v1.MonitorAccountRequest,
    stream v1.AccountService_MonitorAccountServer,
) error {
    eventCh, unsubscribe := s.accountSvc.SubscribeToEvents(req.GetAccountId())
    defer unsubscribe()

    for {
        select {
        case <-stream.Context().Done():
            return stream.Context().Err()
        case event, ok := <-eventCh:
            if !ok {
                return nil
            }
            protoEvent := &v1.AccountEvent{
                EventId:   event.ID,
                AccountId: event.AccountID,
                Type:      toProtoEventType(event.Type),
                Details:   event.Details,
                Timestamp: timestamppb.New(event.Timestamp),
            }
            if err := stream.Send(protoEvent); err != nil {
                return err
            }
        }
    }
}
```

In a Citi microservices architecture, unary RPCs handle most CRUD operations, server streaming handles reporting, client streaming handles bulk imports, and bidirectional streaming handles real-time notifications and event sourcing.

**Key Points to Hit:**
- [ ] Unary: request/response (standard CRUD)
- [ ] Server streaming: one request, many responses (large result sets)
- [ ] Client streaming: many requests, one response (bulk uploads)
- [ ] Bidirectional: both sides stream (real-time events, notifications)
- [ ] Each streaming type handles `stream.Context().Done()` for cancellation

**Follow-Up Questions:**
1. How do you handle backpressure in server streaming?
2. What happens to a streaming connection during a server deploy?
3. How do you add authentication to gRPC streaming calls?

**Source:** `backend-engineering/go/grpc.md`

---

### Q12: 🟡 How do you configure a gRPC server for production banking use, including keepalive, message limits, and reflection?

**Strong Answer:**

A production gRPC server requires careful configuration to handle connection lifecycle, resource limits, and debuggability:

```go
srv := grpc.NewServer(
    // Keepalive: detect and clean up dead connections
    grpc.KeepaliveParams(keepalive.ServerParameters{
        MaxConnectionIdle:     5 * time.Minute,   // Close connections idle > 5m
        MaxConnectionAge:      10 * time.Minute,  // Force reconnect every 10m
        MaxConnectionAgeGrace: 5 * time.Minute,   // Allow in-flight calls to finish
    }),
    grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
        MinTime:             10 * time.Second,    // Reject pings more frequent than 10s
        PermitWithoutStream: true,                // Allow pings without active streams
    }),
    // Message size limits (prevent memory exhaustion)
    grpc.MaxRecvMsgSize(10 * 1024 * 1024),  // 10MB max receive
    grpc.MaxSendMsgSize(10 * 1024 * 1024),  // 10MB max send
)

// Register services
v1.RegisterAccountServiceServer(srv, accountServer)

// Enable reflection for debugging with grpcurl/grpcui
reflection.Register(srv)

lis, _ := net.Listen("tcp", ":50051")
srv.Serve(lis)
```

**Keepalive** is critical in banking microservices because load balancers and firewalls silently drop idle TCP connections. Without keepalive, clients hang on "zombie" connections. `MaxConnectionAge` forces periodic reconnection, which also provides natural load rebalancing.

**Message size limits** prevent a single request from consuming all available memory. The default is 4MB; 10MB is reasonable for most banking operations.

**Reflection** enables tools like `grpcurl` to discover service definitions without the `.proto` files:
```bash
grpcurl -plaintext localhost:50051 list
grpcurl -plaintext localhost:50051 describe api.v1.AccountService
```

**Key Points to Hit:**
- [ ] Keepalive parameters prevent zombie connections from accumulating
- [ ] MaxConnectionAge forces periodic reconnection (also aids load balancing)
- [ ] MaxRecvMsgSize/MaxSendMsgSize prevent memory exhaustion
- [ ] Reflection enables debugging without distributing .proto files
- [ ] In production, TLS must be enabled (the example uses insecure for simplicity)

**Follow-Up Questions:**
1. How do you add TLS to a gRPC server?
2. What is the difference between keepalive server parameters and enforcement policy?
3. How do you add interceptors for logging and metrics?

**Source:** `backend-engineering/go/grpc.md`

---

### Q13: 🟡 How do you profile a Go service in production to identify CPU and memory bottlenecks?

**Strong Answer:**

Go's built-in `net/http/pprof` package exposes profiling endpoints that can be queried without restarting the service. The standard approach is to run pprof on a separate port accessible only internally:

```go
func main() {
    // Start pprof on internal port
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()
    runApp()
}
```

**CPU profiling** — identifies functions consuming the most CPU time:
```bash
# Capture 30-second CPU profile
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Interactive analysis
(pprof) top 10          # Top 10 functions by CPU time
(pprof) list main       # Show source with per-line annotations
(pprof) web             # Generate call graph SVG
```

**Memory profiling** — identifies allocation hotspots:
```bash
# Current heap profile
go tool pprof http://localhost:6060/debug/pprof/heap

# Compare two profiles to find leaks
go tool pprof -base heap.before heap.after
```

**Goroutine profiling** — finds goroutine leaks:
```bash
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

In a banking payment gateway with high latency, the diagnostic workflow would be:
1. Check CPU profile to find hot functions (e.g., JSON serialization, cryptographic operations)
2. Check heap profile to find allocation hotspots
3. Check goroutine profile to detect leaked goroutines
4. Use `go test -bench=. -benchmem` to benchmark suspected functions locally

```go
// Benchmark a suspected hot path
func BenchmarkJSONMarshal(b *testing.B) {
    payment := Payment{
        ID:     "PAY-001",
        Amount: decimal.NewFromInt(100),
        Status: "pending",
    }
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        _, _ = json.Marshal(payment)
    }
}
// Run: go test -bench=BenchmarkJSONMarshal -benchmem
// Output: BenchmarkJSONMarshal-8   1000000   500 ns/op   200 B/op   5 allocs/op
```

**Key Points to Hit:**
- [ ] `net/http/pprof` exposes profiling endpoints on a dedicated port
- [ ] CPU profile identifies functions consuming the most CPU
- [ ] Heap profile identifies allocation hotspots and potential leaks
- [ ] Goroutine profile detects goroutine leaks
- [ ] Benchmarks with `-benchmem` track allocations per operation

**Follow-Up Questions:**
1. How do you profile a Go service in production without impacting performance?
2. What does "escape analysis" tell you and how do you view it?
3. How do you use `go tool pprof -diff` to compare profiles before and after a change?

**Source:** `backend-engineering/go/performance.md`

---

### Q14: 🟡 How do you reduce memory allocations in a Go service's hot path?

**Strong Answer:**

Memory allocations in hot paths directly impact latency because they trigger garbage collection pauses. Three key optimization strategies:

**1. Use `strings.Builder` instead of string concatenation:**
```go
// BAD: creates a new string each iteration
func BuildErrorMessage(fields []string) string {
    msg := ""
    for _, f := range fields {
        msg += f + ", "  // New allocation each time
    }
    return msg
}

// GOOD: reuses a single buffer
func BuildErrorMessageOptimized(fields []string) string {
    var sb strings.Builder
    for i, f := range fields {
        if i > 0 {
            sb.WriteString(", ")
        }
        sb.WriteString(f)
    }
    return sb.String()
}
// Benchmark: 200 B/op, 5 allocs → 32 B/op, 1 alloc
```

**2. Pre-allocate slices when capacity is known:**
```go
// BAD: slice grows exponentially (2x, 4x, 8x...), reallocating each time
func ProcessPayments(payments []Payment) []Result {
    var results []Result
    for _, p := range payments {
        results = append(results, processOne(p))
    }
    return results
}

// GOOD: single allocation
func ProcessPaymentsOptimized(payments []Payment) []Result {
    results := make([]Result, 0, len(payments))
    for _, p := range payments {
        results = append(results, processOne(p))
    }
    return results
}
```

**3. Use `sync.Pool` for frequently allocated temporary objects:**
```go
var bufferPool = sync.Pool{
    New: func() any {
        return new(bytes.Buffer)
    },
}

func ProcessRequest(data []byte) []byte {
    buf := bufferPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufferPool.Put(buf)
    }()
    buf.Write(data)
    // Process...
    return buf.Bytes()
}
```

In a banking API gateway handling 10,000 requests/second, these optimizations compound. Reducing allocations from 50 per request to 10 means 40,000 fewer allocations per second, which directly reduces GC pause times and improves p99 latency.

**Key Points to Hit:**
- [ ] `strings.Builder` reuses a buffer instead of creating new strings
- [ ] Pre-allocating slices with `make([]T, 0, cap)` avoids reallocation during growth
- [ ] `sync.Pool` recycles temporary objects (buffers, request structs)
- [ ] Every allocation triggers GC work — fewer allocations = shorter GC pauses
- [ ] Benchmark with `-benchmem` to measure allocations per operation

**Follow-Up Questions:**
1. What is escape analysis and how does it relate to allocation?
2. When is `sync.Pool` counterproductive?
3. How do you choose between `json.Marshal` and a faster alternative like `jsoniter`?

**Source:** `backend-engineering/go/performance.md`

---

### Q15: 🟡 How do you write table-driven tests for a payment validation function, and how do you test error conditions?

**Strong Answer:**

Table-driven tests define all test cases as a slice of structs, then iterate over them with `t.Run()`. This pattern is idiomatic Go and scales cleanly to dozens of cases:

```go
func TestValidatePayment(t *testing.T) {
    tests := []struct {
        name        string
        payment     model.CreatePaymentRequest
        expectedErr error
    }{
        {
            name: "valid payment",
            payment: model.CreatePaymentRequest{
                FromAccount: "ACC-001",
                ToAccount:   "ACC-002",
                Amount:      decimal.NewFromInt(100),
                Currency:    "USD",
            },
            expectedErr: nil,
        },
        {
            name: "zero amount rejected",
            payment: model.CreatePaymentRequest{
                FromAccount: "ACC-001",
                ToAccount:   "ACC-002",
                Amount:      decimal.Zero,
                Currency:    "USD",
            },
            expectedErr: model.ErrInvalidInput,
        },
        {
            name: "negative amount rejected",
            payment: model.CreatePaymentRequest{
                FromAccount: "ACC-001",
                ToAccount:   "ACC-002",
                Amount:      decimal.NewFromInt(-100),
                Currency:    "USD",
            },
            expectedErr: model.ErrInvalidInput,
        },
        {
            name: "self payment rejected",
            payment: model.CreatePaymentRequest{
                FromAccount: "ACC-001",
                ToAccount:   "ACC-001",
                Amount:      decimal.NewFromInt(100),
                Currency:    "USD",
            },
            expectedErr: model.ErrSelfPayment,
        },
        {
            name: "unsupported currency rejected",
            payment: model.CreatePaymentRequest{
                FromAccount: "ACC-001",
                ToAccount:   "ACC-002",
                Amount:      decimal.NewFromInt(100),
                Currency:    "XYZ",
            },
            expectedErr: model.ErrInvalidInput,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := ValidatePayment(tt.payment)

            if tt.expectedErr == nil {
                if err != nil {
                    t.Errorf("expected no error, got %v", err)
                }
            } else {
                if err == nil {
                    t.Errorf("expected error %v, got nil", tt.expectedErr)
                } else if !errors.Is(err, tt.expectedErr) {
                    t.Errorf("expected error %v, got %v", tt.expectedErr, err)
                }
            }
        })
    }
}
```

For **testing service methods that depend on repositories**, use interface-based mocking:

```go
type mockPaymentRepo struct {
    payments map[string]*model.Payment
    err      error
}

func (m *mockPaymentRepo) GetByID(ctx context.Context, id string) (*model.Payment, error) {
    if m.err != nil {
        return nil, m.err
    }
    p, ok := m.payments[id]
    if !ok {
        return nil, model.ErrNotFound
    }
    return p, nil
}

func TestPaymentService_CreatePayment(t *testing.T) {
    tests := []struct {
        name    string
        repo    *mockPaymentRepo
        wantErr bool
    }{
        {
            name:    "repository error",
            repo:    &mockPaymentRepo{err: fmt.Errorf("database connection failed")},
            wantErr: true,
        },
        {
            name:    "successful creation",
            repo:    &mockPaymentRepo{payments: make(map[string]*model.Payment)},
            wantErr: false,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            svc := service.NewPaymentService(tt.repo, nil)
            _, err := svc.CreatePayment(context.Background(), model.CreatePaymentCommand{
                FromAccount: "ACC-001",
                ToAccount:   "ACC-002",
                Amount:      decimal.NewFromInt(100),
                Currency:    "USD",
            })
            if tt.wantErr && err == nil {
                t.Error("expected error, got nil")
            }
        })
    }
}
```

**Key Points to Hit:**
- [ ] Table-driven tests: slice of structs with `name`, inputs, and expected outputs
- [ ] `t.Run(name, func)` creates a subtest per case with clear output
- [ ] Use `errors.Is()` to check wrapped sentinel errors in assertions
- [ ] Mock dependencies via interfaces — replace repo with a test double
- [ ] `nil` logger in tests keeps output clean

**Follow-Up Questions:**
1. How do you test HTTP handlers that make external API calls?
2. When would you use fuzzing instead of table-driven tests?
3. How do you test concurrent code?

**Source:** `backend-engineering/go/testing.md`

---

### Q16: 🔴 How do you perform escape analysis in Go, and how does it inform allocation optimization decisions?

**Strong Answer:**

Escape analysis is a compile-time process that determines whether a variable's lifetime extends beyond the function that creates it. If a variable "escapes" to the heap, it becomes a garbage-collected allocation. If it does not escape, it stays on the stack (which is free — stack memory is reclaimed when the function returns).

View escape analysis output with the `-gcflags='-m'` flag:

```bash
go build -gcflags='-m' ./cmd/server 2>&1 | grep "escapes"
```

Example output:
```
./payment.go:25:10: p escapes to heap
./payment.go:30:12: string(f) escapes to heap
```

**Common escape scenarios:**
- Returning a pointer to a local variable
- Storing a value in a global variable or interface
- Sending a value through a channel (the receiver may outlive the sender)
- Appending to a slice that escapes

**Optimization strategy:**

```go
// ESCAPES: returning pointer to struct allocated in function
func NewPayment() *Payment {
    p := Payment{ID: "PAY-001"}
    return &p  // p escapes to heap
}

// Does NOT escape: return by value
func NewPayment() Payment {
    return Payment{ID: "PAY-001"}  // stays on stack, copied to caller
}

// ESCAPES: string concatenation in interface argument
fmt.Printf("Payment: %s\n", p.ID)  // p.ID escapes because fmt.Printf takes interface{}

// Does NOT escape: use strings.Builder to build string before passing
```

In a high-throughput banking API, reducing escapes means fewer heap allocations, which means less GC work per request. The goal is not to eliminate all escapes (some are unavoidable and correct), but to ensure that allocations in the hot path are intentional.

**Key Points to Hit:**
- [ ] Escape analysis determines if a variable lives on the stack or heap
- [ ] `go build -gcflags='-m'` shows escape analysis output
- [ ] Stack allocations are free; heap allocations trigger GC work
- [ ] Pointers returned from functions, interface arguments, and channel sends cause escapes
- [ ] In banking hot paths, minimizing escapes reduces GC pause times and improves p99 latency

**Follow-Up Questions:**
1. When is it actually fine for a variable to escape?
2. How do you use `go test -gcflags='-m'` to analyze test code?
3. What is the relationship between escape analysis and `sync.Pool`?

**Source:** `backend-engineering/go/performance.md`

---

### Q17: 🔴 How do you test concurrent code in Go, including goroutine leak detection and race condition testing?

**Strong Answer:**

Testing concurrent code requires three strategies: **race detection**, **leak detection**, and **deterministic testing** of non-deterministic behavior.

**1. Race detection with the race detector:**
```bash
go test -race ./...
```
The `-race` flag instruments all memory accesses at runtime to detect data races. It should be run in CI on every commit. In banking code, a data race on a shared balance map could cause incorrect account balances — a regulatory violation.

**2. Goroutine leak detection:**
```go
import "go.uber.org/goleak"

func TestPaymentWorker(t *testing.T) {
    defer goleak.VerifyNone(t)  // Fails if goroutines leak

    worker := NewPaymentWorker(10)
    worker.Start()

    // Submit work
    for i := 0; i < 5; i++ {
        worker.Submit(Payment{ID: fmt.Sprintf("PAY-%d", i)})
    }

    worker.Stop()
    // goleak checks that all goroutines have exited
}
```

**3. Deterministic testing with channels and synchronization:**
```go
func TestWorkerPoolProcessesAllJobs(t *testing.T) {
    worker := NewPaymentWorker(3)
    worker.Start()

    expected := 10
    for i := 0; i < expected; i++ {
        worker.Submit(Payment{ID: fmt.Sprintf("PAY-%d", i)})
    }

    // Collect results with timeout to prevent infinite hang
    var results []PaymentResult
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    for i := 0; i < expected; i++ {
        select {
        case result := <-worker.resultQueue:
            results = append(results, result)
        case <-ctx.Done():
            t.Fatalf("timeout: got %d results, expected %d", len(results), expected)
        }
    }

    if len(results) != expected {
        t.Errorf("got %d results, want %d", len(results), expected)
    }

    worker.Stop()
}
```

**Key testing rules for concurrency:**
- Always run `go test -race` in CI
- Use `goleak` to detect goroutine leaks
- Add timeouts (`context.WithTimeout`) to all tests that wait on channels
- Test the public behavior (all jobs processed, no panics), not internal goroutine ordering

**Key Points to Hit:**
- [ ] `-race` flag detects data races at runtime — must be in CI
- [ ] `goleak.VerifyNone(t)` detects goroutine leaks
- [ ] Always add timeouts to channel-based tests to prevent infinite hangs
- [ ] Test observable outcomes (all results received) not internal ordering
- [ ] Data races in banking can cause incorrect financial calculations

**Follow-Up Questions:**
1. How do you test a worker pool that processes jobs from a Kafka topic?
2. What is the difference between a data race and a race condition?
3. How do you reproduce a race condition that only appears under load?

**Source:** `backend-engineering/go/testing.md`, `backend-engineering/go/concurrency.md`

---

### Q18: 🔴 Design an HTTP handler testing strategy for a banking payment endpoint that covers success, validation errors, service errors, and integration with a real database.

**Strong Answer:**

A comprehensive testing strategy uses **multiple layers of tests**, from fast unit tests to slower integration tests:

**Layer 1 — Unit test handler with mocked service (httptest):**
```go
func TestPaymentHandler_CreatePayment_Success(t *testing.T) {
    mockSvc := &mockPaymentService{
        createPaymentFunc: func(ctx context.Context, cmd model.CreatePaymentCommand) (*model.Payment, error) {
            return &model.Payment{
                ID:     "PAY-001",
                Status: model.PaymentStatusPending,
                Amount: cmd.Amount,
            }, nil
        },
    }

    handler := handler.NewPaymentHandler(mockSvc, zaptest.NewLogger(t))

    body, _ := json.Marshal(map[string]interface{}{
        "from_account": "ACC-001",
        "to_account":   "ACC-002",
        "amount":       "100.00",
        "currency":     "USD",
    })
    req := httptest.NewRequest(http.MethodPost, "/v1/payments", bytes.NewReader(body))
    req.Header.Set("Content-Type", "application/json")

    rr := httptest.NewRecorder()
    handler.CreatePayment(rr, req)

    if rr.Code != http.StatusCreated {
        t.Errorf("expected status %d, got %d", http.StatusCreated, rr.Code)
    }

    var resp map[string]interface{}
    json.Unmarshal(rr.Body.Bytes(), &resp)
    if resp["id"] != "PAY-001" {
        t.Errorf("expected id PAY-001, got %v", resp["id"])
    }
}
```

**Layer 2 — Test validation errors:**
```go
func TestPaymentHandler_CreatePayment_ValidationError(t *testing.T) {
    mockSvc := &mockPaymentService{}
    handler := handler.NewPaymentHandler(mockSvc, zaptest.NewLogger(t))

    body, _ := json.Marshal(map[string]interface{}{
        "from_account": "ACC-001",
        // missing to_account, amount, currency
    })
    req := httptest.NewRequest(http.MethodPost, "/v1/payments", bytes.NewReader(body))
    req.Header.Set("Content-Type", "application/json")

    rr := httptest.NewRecorder()
    handler.CreatePayment(rr, req)

    if rr.Code != http.StatusBadRequest {
        t.Errorf("expected status %d, got %d", http.StatusBadRequest, rr.Code)
    }
}
```

**Layer 3 — Integration test with real database:**
```go
func TestPaymentService_CreatePayment_Integration(t *testing.T) {
    db := testutil.SetupTestDB(t)  // Real Postgres, cleaned up via t.Cleanup()
    repo := repository.NewPaymentRepository(db)
    svc := service.NewPaymentService(repo, nil)

    result, err := svc.CreatePayment(context.Background(), model.CreatePaymentCommand{
        FromAccount: "ACC-001",
        ToAccount:   "ACC-002",
        Amount:      decimal.NewFromInt(100),
        Currency:    "USD",
    })

    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }

    // Verify persisted to database
    saved, err := repo.GetByID(context.Background(), result.ID)
    if err != nil {
        t.Fatalf("payment not found in DB: %v", err)
    }
    if !saved.Amount.Equal(decimal.NewFromInt(100)) {
        t.Errorf("expected amount 100, got %s", saved.Amount)
    }
}
```

**The testing pyramid for banking services:**
- Many fast unit tests (handler + mocked service) — runs in < 1 second
- Fewer integration tests (service + real database) — runs in < 10 seconds
- End-to-end tests (full HTTP stack) — run in CI on main branch only

**Key Points to Hit:**
- [ ] `httptest.NewRecorder()` captures HTTP responses without starting a server
- [ ] Mock services via interfaces for fast, isolated unit tests
- [ ] Integration tests use real database with `t.Cleanup()` for teardown
- [ ] Test all error paths: validation, service errors, unexpected failures
- [ ] `zaptest.NewLogger(t)` writes logs only on test failure, keeping output clean

**Follow-Up Questions:**
1. How do you test a handler that calls an external payment gateway?
2. What is the difference between `httptest.NewRecorder()` and `httptest.NewServer()`?
3. How do you seed test data for integration tests?

**Source:** `backend-engineering/go/testing.md`, `backend-engineering/go/service-structure.md`

---

### Q19: 🔴 How do you handle graceful shutdown in a Go banking service, and what happens to in-flight requests?

**Strong Answer:**

Graceful shutdown ensures that when a service is terminated (during deployment, scaling, or failure recovery), in-flight requests complete within a bounded time window rather than being abruptly killed. In banking, abrupt termination during a payment could cause duplicate charges or lost transactions.

```go
func main() {
    // ... setup ...

    srv := server.New(mux, server.Options{
        Addr:         fmt.Sprintf(":%d", cfg.Port),
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 30 * time.Second,
        IdleTimeout:  120 * time.Second,
    })

    // Listen for SIGINT (Ctrl+C) and SIGTERM (Kubernetes/Docker)
    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()

    go func() {
        logger.Info("Starting server", zap.Int("port", cfg.Port))
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            logger.Fatal("Server failed", zap.Error(err))
        }
    }()

    // Block until shutdown signal
    <-ctx.Done()
    logger.Info("Shutting down server...")

    // Give in-flight requests time to complete
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(shutdownCtx); err != nil {
        logger.Fatal("Server shutdown failed", zap.Error(err))
    }

    logger.Info("Server stopped gracefully")
}
```

**What happens during shutdown:**

1. `signal.NotifyContext` receives SIGTERM (from Kubernetes, Docker, or OS)
2. `ctx.Done()` unblocks, triggering the shutdown sequence
3. `srv.Shutdown(shutdownCtx)` stops accepting new connections
4. In-flight requests continue processing, bounded by the 30-second timeout
5. If any request exceeds the timeout, it is forcefully cancelled via context
6. The process exits cleanly

**Critical banking considerations:**

- The shutdown timeout (30s) must exceed your slowest expected operation (e.g., a payment that calls an external gateway with a 25-second timeout)
- Database connections must be closed: `defer db.Close()` ensures cleanup
- Background goroutines (workers, schedulers) must listen to the same context
- Kubernetes sends SIGTERM, waits for the `terminationGracePeriodSeconds`, then sends SIGKILL

```go
// Worker pool respecting shutdown context
func (w *PaymentWorker) Start() {
    for i := 0; i < w.workerCount; i++ {
        go func(id int) {
            for {
                select {
                case <-w.ctx.Done():
                    log.Printf("Worker %d: draining current job before exit", id)
                    return
                case payment := <-w.jobQueue:
                    processPayment(payment)
                }
            }
        }(i)
    }
}
```

**Key Points to Hit:**
- [ ] `signal.NotifyContext` listens for SIGINT and SIGTERM
- [ ] `srv.Shutdown(ctx)` stops accepting new connections but drains in-flight requests
- [ ] Shutdown timeout must exceed the longest operation's timeout
- [ ] Kubernetes sends SIGTERM then SIGKILL after terminationGracePeriodSeconds
- [ ] All background goroutines must respect the shutdown context

**Follow-Up Questions:**
1. How do you handle database transactions that are in-flight during shutdown?
2. What is the difference between `srv.Close()` and `srv.Shutdown(ctx)`?
3. How do you test graceful shutdown in automated tests?

**Source:** `backend-engineering/go/service-structure.md`

---

### Q20: 🔴 How would you architect a Go service that proxies concurrent LLM inference requests for a GenAI banking platform?

**Strong Answer:**

A GenAI proxy service in Go sits between banking clients (internal apps, APIs) and Python-based LLM microservices. Go handles the high-concurrency API gateway layer while Python handles model inference. The design leverages Go's strengths in concurrent I/O, gRPC, and performance.

**Architecture:**

```
Client → Go API Gateway → Python LLM Service (gRPC)
               ↓
         Redis (caching, rate limiting)
               ↓
         Audit Logger (async, buffered channel)
```

**Key design decisions:**

```go
// 1. Concurrent request handling with bounded worker pool
type GenAIProxy struct {
    llmClient    pb.LLMServiceClient
    cache        *redis.Client
    rateLimiter  *RateLimiter
    auditCh      chan AuditEvent  // Buffered: async audit logging
    workerCount  int
}

func (p *GenAIProxy) HandlePrompt(ctx context.Context, req PromptRequest) (*PromptResponse, error) {
    // Rate limit per user
    if !p.rateLimiter.Allow(req.UserID) {
        return nil, model.ErrRateLimited
    }

    // Check cache
    cached, err := p.cache.Get(ctx, req.CacheKey()).Result()
    if err == nil {
        return &PromptResponse{Text: cached, Source: "cache"}, nil
    }

    // Proxy to LLM service with timeout
    ctx, cancel := context.WithTimeout(ctx, 60*time.Second)
    defer cancel()

    resp, err := p.llmClient.Generate(ctx, &pb.GenerateRequest{
        Prompt:   req.Prompt,
        MaxTokens: req.MaxTokens,
        UserId:   req.UserID,
    })
    if err != nil {
        // Log audit event asynchronously (non-blocking)
        select {
        case p.auditCh <- AuditEvent{Type: "llm_error", UserID: req.UserID, Err: err}:
        default:
            // Audit buffer full — log directly
        }
        return nil, fmt.Errorf("calling LLM service: %w", err)
    }

    // Cache response
    p.cache.Set(ctx, req.CacheKey(), resp.Text, 5*time.Minute)

    // Async audit
    select {
    case p.auditCh <- AuditEvent{Type: "llm_success", UserID: req.UserID, Tokens: resp.TokensUsed}:
    default:
    }

    return &PromptResponse{Text: resp.Text, Source: "llm"}, nil
}
```

**Why Go for this role:**
- **Concurrency**: Each LLM request takes 5-30 seconds. Go can handle thousands of concurrent long-running requests with goroutines (2 KB each), whereas Java would need 1-2 MB per thread.
- **gRPC**: Type-safe, efficient communication with Python LLM services via protobuf contracts.
- **Low memory footprint**: 10-50 MB per Go instance vs 100-500 MB for Java/Python, enabling dense Kubernetes deployments.
- **Fast startup**: < 100ms startup enables rapid scaling in response to traffic spikes.

**Why not Go for inference:**
- Go lacks the ML ecosystem (no PyTorch, TensorFlow equivalents). Python microservices handle model inference; Go handles the orchestration, caching, rate limiting, and audit.

**Banking-specific considerations:**
- Audit events are captured for every LLM interaction (regulatory requirement for AI usage tracking)
- Rate limiting prevents abuse and controls LLM API costs
- Response caching reduces redundant LLM calls for common queries
- Deadlines/timeouts prevent runaway LLM calls from consuming resources

**Key Points to Hit:**
- [ ] Go acts as API gateway/orchestrator; Python handles model inference
- [ ] Goroutines handle thousands of concurrent long-running LLM requests efficiently
- [ ] gRPC provides type-safe, low-latency communication with Python LLM services
- [ ] Async audit logging via buffered channels captures every interaction for compliance
- [ ] Rate limiting + caching controls costs and prevents abuse

**Follow-Up Questions:**
1. How would you implement streaming LLM responses (server-sent events) through the Go proxy?
2. How do you handle LLM service failures — retry, circuit breaker, or fallback?
3. What metrics would you expose for monitoring the GenAI proxy in production?

**Source:** `backend-engineering/go/README.md`, `backend-engineering/go/concurrency.md`, `backend-engineering/go/grpc.md`, `backend-engineering/go/service-structure.md`
