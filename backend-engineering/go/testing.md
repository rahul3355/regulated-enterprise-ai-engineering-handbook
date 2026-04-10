# Go Testing Patterns

## Testing Philosophy

Go's built-in testing package with table-driven tests provides a simple, powerful testing approach. No external framework needed. Tests are co-located with code, run with `go test`, and produce clear output.

## Table-Driven Tests

```go
// internal/service/payment_test.go
package service

import (
    "context"
    "testing"
    "decimal"

    "github.com/bank/payment-service/internal/model"
)

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

## Mocking with Interfaces

```go
// internal/service/payment_test.go

// Mock repository for testing
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

func (m *mockPaymentRepo) Create(ctx context.Context, p *model.Payment) error {
    if m.err != nil {
        return m.err
    }
    m.payments[p.ID] = p
    return nil
}

func TestPaymentService_CreatePayment(t *testing.T) {
    tests := []struct {
        name        string
        repo        *mockPaymentRepo
        cmd         model.CreatePaymentCommand
        wantErr     bool
        checkResult func(t *testing.T, result *model.Payment)
    }{
        {
            name: "successful payment creation",
            repo: &mockPaymentRepo{
                payments: make(map[string]*model.Payment),
            },
            cmd: model.CreatePaymentCommand{
                FromAccount: "ACC-001",
                ToAccount:   "ACC-002",
                Amount:      decimal.NewFromInt(100),
                Currency:    "USD",
            },
            wantErr: false,
            checkResult: func(t *testing.T, result *model.Payment) {
                if result.Status != model.PaymentStatusPending {
                    t.Errorf("expected status pending, got %s", result.Status)
                }
                if !result.Amount.Equal(decimal.NewFromInt(100)) {
                    t.Errorf("expected amount 100, got %s", result.Amount)
                }
            },
        },
        {
            name: "repository error",
            repo: &mockPaymentRepo{
                err: fmt.Errorf("database connection failed"),
            },
            cmd: model.CreatePaymentCommand{
                FromAccount: "ACC-001",
                ToAccount:   "ACC-002",
                Amount:      decimal.NewFromInt(100),
                Currency:    "USD",
            },
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            svc := NewPaymentService(tt.repo, nil) // nil logger for test
            result, err := svc.CreatePayment(context.Background(), tt.cmd)

            if tt.wantErr {
                if err == nil {
                    t.Error("expected error, got nil")
                }
                return
            }

            if err != nil {
                t.Errorf("unexpected error: %v", err)
                return
            }

            if tt.checkResult != nil {
                tt.checkResult(t, result)
            }
        })
    }
}
```

## HTTP Handler Testing

```go
// internal/handler/payment_test.go
package handler

import (
    "bytes"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"

    "go.uber.org/zap/zaptest"
)

func TestPaymentHandler_CreatePayment(t *testing.T) {
    // Mock service
    mockSvc := &mockPaymentService{
        createPaymentFunc: func(ctx context.Context, cmd model.CreatePaymentCommand) (*model.Payment, error) {
            return &model.Payment{
                ID:     "PAY-001",
                Status: model.PaymentStatusPending,
                Amount: cmd.Amount,
            }, nil
        },
    }

    handler := NewPaymentHandler(mockSvc, zaptest.NewLogger(t))

    // Create request
    body, _ := json.Marshal(map[string]interface{}{
        "from_account": "ACC-001",
        "to_account":   "ACC-002",
        "amount":       "100.00",
        "currency":     "USD",
    })
    req := httptest.NewRequest(http.MethodPost, "/v1/payments", bytes.NewReader(body))
    req.Header.Set("Content-Type", "application/json")

    // Record response
    rr := httptest.NewRecorder()
    handler.CreatePayment(rr, req)

    // Assert response
    if rr.Code != http.StatusCreated {
        t.Errorf("expected status %d, got %d", http.StatusCreated, rr.Code)
    }

    var resp map[string]interface{}
    if err := json.Unmarshal(rr.Body.Bytes(), &resp); err != nil {
        t.Fatalf("failed to unmarshal response: %v", err)
    }

    if resp["id"] != "PAY-001" {
        t.Errorf("expected id PAY-001, got %v", resp["id"])
    }
}

func TestPaymentHandler_CreatePayment_ValidationError(t *testing.T) {
    mockSvc := &mockPaymentService{}
    handler := NewPaymentHandler(mockSvc, zaptest.NewLogger(t))

    // Invalid request (missing required fields)
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

## Integration Testing

```go
// testutil/testdb.go
package testutil

import (
    "database/sql"
    "testing"
)

// SetupTestDB creates a test database and returns cleanup function
func SetupTestDB(t *testing.T) *sql.DB {
    t.Helper()

    dsn := "postgresql://test:test@localhost:5432/banking_test"
    db, err := sql.Open("pgx", dsn)
    if err != nil {
        t.Fatalf("failed to open database: %v", err)
    }

    // Create tables
    if err := runMigrations(db); err != nil {
        t.Fatalf("failed to run migrations: %v", err)
    }

    // Cleanup
    t.Cleanup(func() {
        // Drop all tables
        dropAllTables(db)
        db.Close()
    })

    return db
}
```

## Benchmarking

```go
func BenchmarkValidatePayment(b *testing.B) {
    payment := model.CreatePaymentRequest{
        FromAccount: "ACC-001",
        ToAccount:   "ACC-002",
        Amount:      decimal.NewFromInt(100),
        Currency:    "USD",
    }

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        _ = ValidatePayment(payment)
    }
}

func BenchmarkCreatePayment(b *testing.B) {
    db := setupTestDB(b)
    repo := repository.NewPaymentRepository(db)
    svc := service.NewPaymentService(repo, nil)

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        _, err := svc.CreatePayment(context.Background(), model.CreatePaymentCommand{
            FromAccount: "ACC-001",
            ToAccount:   "ACC-002",
            Amount:      decimal.NewFromInt(100),
            Currency:    "USD",
        })
        if err != nil {
            b.Fatal(err)
        }
    }
}

// Run benchmarks:
// go test -bench=. -benchmem
// go test -bench=. -benchmem -cpuprofile=cpu.prof
```

## Test Coverage

```bash
# Run tests with coverage
go test ./... -coverprofile=coverage.out

# View coverage in terminal
go tool cover -func=coverage.out

# View coverage in browser
go tool cover -html=coverage.out

# Minimum coverage threshold
go test ./... -coverprofile=coverage.out
# Check: go tool cover -func=coverage.out | grep total | awk '{print $3}'
```

## Fuzzing (Go 1.18+)

```go
func FuzzValidatePayment(f *testing.F) {
    // Seed corpus
    f.Add("ACC-001", "ACC-002", "100.00", "USD")
    f.Add("ACC-003", "ACC-004", "0.01", "EUR")

    f.Fuzz(func(t *testing.T, from, to, amount, currency string) {
        amt, err := decimal.NewFromString(amount)
        if err != nil {
            return // Skip invalid amounts
        }

        payment := model.CreatePaymentRequest{
            FromAccount: from,
            ToAccount:   to,
            Amount:      amt,
            Currency:    currency,
        }

        // Should not panic
        _ = ValidatePayment(payment)
    })
}

// Run fuzzing:
// go test -fuzz=Fuzz -fuzztime=30s
```

## Common Mistakes

1. **Not using table-driven tests**: Writing separate test functions for each case
2. **Testing private functions**: Tests should test public API, not implementation
3. **No test isolation**: Tests sharing state interfere with each other
4. **Ignoring t.Helper()**: Test helper functions should mark themselves as helpers
5. **Not using t.Cleanup()**: Manual cleanup instead of defer-based cleanup
6. **No benchmarks**: Not measuring performance of critical paths
7. **Mocking too much**: Testing mocks instead of real behavior

## Interview Questions

1. How do you test a handler that depends on a database and external API?
2. Design table-driven tests for a payment validation function with 10 cases.
3. How do you benchmark a function and identify memory allocations?
4. What is the difference between unit and integration tests in Go?
5. How do you fuzz test an input validation function?

## Cross-References

- `./service-structure.md` - Testing handlers and services
- `./error-handling.md` - Testing error conditions
- `./concurrency.md` - Testing concurrent code
- `../backend-testing/README.md` - Overall testing strategy
- `../api-design/README.md` - Testing API contracts
