# Go Performance Engineering

## Profiling

### CPU Profiling

```go
import (
    "net/http"
    _ "net/http/pprof"
    "runtime/pprof"
    "os"
)

func main() {
    // Enable pprof endpoints
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()

    // Access profiles:
    // CPU:     http://localhost:6060/debug/pprof/profile?seconds=30
    // Memory:  http://localhost:6060/debug/pprof/heap
    // Goroutine: http://localhost:6060/debug/pprof/goroutine
    // Block:   http://localhost:6060/debug/pprof/block

    runApp()
}
```

```bash
# Collect CPU profile
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Interactive analysis
(pprof) top 10          # Top 10 functions by CPU time
(pprof) list main       # Show source code with annotations
(pprof) web             # Generate call graph SVG

# Save to file
go tool pprof -svg http://localhost:6060/debug/pprof/profile > profile.svg
```

### Memory Profiling

```go
// Manual memory profile
func writeHeapProfile(filename string) {
    f, err := os.Create(filename)
    if err != nil {
        log.Fatal(err)
    }
    defer f.Close()

    runtime.GC() // Force GC for accurate stats
    pprof.WriteHeapProfile(f)
}

// Analyze
go tool pprof -http=:8080 heap.prof
```

### Benchmarking

```go
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

func BenchmarkJSONMarshalOptimized(b *testing.B) {
    payment := Payment{
        ID:     "PAY-001",
        Amount: decimal.NewFromInt(100),
        Status: "pending",
    }

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        _, _ = jsoniter.Marshal(payment)  // Faster JSON library
    }
}

// Run with allocation tracking
// go test -bench=. -benchmem
```

## Allocation Optimization

### Avoid Unnecessary Allocations

```go
// BAD: String concatenation creates new strings
func BuildErrorMessage(fields []string) string {
    msg := ""
    for _, f := range fields {
        msg += f + ", "  // New string each iteration
    }
    return msg
}

// GOOD: strings.Builder reuses buffer
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

// Benchmark results:
// BuildErrorMessage:          1000000 ops, 500 ns/op, 200 B/op, 5 allocs/op
// BuildErrorMessageOptimized: 5000000 ops, 100 ns/op, 32 B/op, 1 allocs/op
```

### Pre-allocate Slices

```go
// BAD: Slice grows dynamically, reallocates
func ProcessPayments(payments []Payment) []Result {
    var results []Result
    for _, p := range payments {
        results = append(results, processOne(p))  // Reallocates as grows
    }
    return results
}

// GOOD: Pre-allocate with known capacity
func ProcessPaymentsOptimized(payments []Payment) []Result {
    results := make([]Result, 0, len(payments))  // Pre-allocate
    for _, p := range payments {
        results = append(results, processOne(p))
    }
    return results
}
```

### Object Pooling

```go
// Pool for frequently allocated objects
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

## HTTP Performance

### Connection Pooling

```go
// Optimized HTTP client
var httpClient = &http.Client{
    Timeout: 30 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 10,
        IdleConnTimeout:     90 * time.Second,
        TLSHandshakeTimeout: 10 * time.Second,
    },
}
```

### Response Body Handling

```go
// BAD: Not limiting response size
resp, err := httpClient.Get(url)
body, err := io.ReadAll(resp.Body)  // Could be gigabytes!

// GOOD: Limit response size
resp, err := httpClient.Get(url)
body, err := io.ReadAll(io.LimitReader(resp.Body, 10*1024*1024))  // 10MB max
```

### Streaming Responses

```go
// Stream large responses instead of loading into memory
func DownloadStatement(w http.ResponseWriter, r *http.Request) {
    // Stream from database
    rows, err := db.QueryContext(r.Context(), "SELECT * FROM statements WHERE account_id = $1", accountID)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    defer rows.Close()

    // Write directly to response
    w.Header().Set("Content-Type", "application/pdf")
    for rows.Next() {
        var data []byte
        if err := rows.Scan(&data); err != nil {
            log.Printf("scan error: %v", err)
            continue
        }
        w.Write(data)
    }
}
```

## Database Performance

### Query Optimization

```go
// Use prepared statements (connection pooling handles this)
stmt, err := db.PrepareContext(ctx, "SELECT balance FROM accounts WHERE id = $1")
if err != nil {
    return err
}
defer stmt.Close()

var balance decimal.Decimal
err = stmt.QueryRowContext(ctx, accountID).Scan(&balance)
```

### Connection Pool Monitoring

```go
// Monitor connection pool stats
func logDBStats(db *sql.DB) {
    stats := db.Stats()
    log.Printf("DB connections: open=%d, in_use=%d, idle=%d, wait_count=%d",
        stats.OpenConnections,
        stats.InUse,
        stats.Idle,
        stats.WaitCount,
    )
}
```

## Memory Management

### Avoiding Memory Leaks

```go
// Goroutine leak: channel never consumed
func leakyFunction() {
    ch := make(chan int)
    go func() {
        ch <- 42  // Blocks forever - no receiver
    }()
}

// Fixed: buffered channel or receiver
func fixedFunction() {
    ch := make(chan int, 1)  // Buffered
    go func() {
        ch <- 42  // Doesn't block
    }()
}

// Context cancellation prevents leaks
func processWithTimeout(ctx context.Context) error {
    ch := make(chan result)

    go func() {
        r := doWork()
        select {
        case ch <- r:
        case <-ctx.Done():
            return
        }
    }()

    select {
    case <-ch:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

## Production Performance Tuning

```go
// GOMAXPROCS: Set to number of CPU cores
runtime.GOMAXPROCS(runtime.NumCPU())

// Garbage collection tuning
debug.SetGCPercent(100)  // Default: trigger GC when heap grows 100%

// For low-latency services (reduce GC pauses)
debug.SetGCPercent(50)   // More frequent, shorter GC pauses

// For throughput-oriented services (fewer GC pauses)
debug.SetGCPercent(200)  // Less frequent, longer GC pauses
```

## Common Mistakes

1. **Not profiling before optimizing**: Optimizing without data
2. **String concatenation in loops**: Using `+` instead of `strings.Builder`
3. **Not pre-allocating slices**: Growing slices causes reallocations
4. **Ignoring connection pool stats**: Not monitoring database connections
5. **Reading unbounded response bodies**: Memory exhaustion from large responses
6. **Goroutine leaks**: Not using context cancellation
7. **No benchmarks**: Not measuring impact of optimizations

## Interview Questions

1. How do you identify the bottleneck in a Go service with high latency?
2. Design a benchmark for a payment validation function.
3. How do you reduce memory allocations in a JSON serialization hot path?
4. What is the difference between `strings.Builder` and string concatenation?
5. How do you profile a Go service in production?

## Cross-References

- `./service-structure.md` - Production service setup
- `./concurrency.md` - Concurrent performance patterns
- `./testing.md` - Benchmarking in Go
- `../performance/README.md` - Overall performance engineering
- `../caching/README.md` - Caching for performance
