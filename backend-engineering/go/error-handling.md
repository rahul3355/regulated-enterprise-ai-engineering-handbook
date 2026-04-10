# Go Error Handling

## Why Error Handling Matters

In banking, unhandled errors cause duplicate payments, lost transactions, and data corruption. Go's explicit error handling forces developers to consider failure cases, but improper error handling still leads to silent failures and cascading issues.

## Sentinel Errors

```go
// internal/model/errors.go
package model

import "errors"

// Sentinel errors for domain-level failures
var (
    ErrNotFound          = errors.New("resource not found")
    ErrUnauthorized      = errors.New("unauthorized")
    ErrInvalidInput      = errors.New("invalid input")
    ErrInsufficientFunds = errors.New("insufficient funds")
    ErrAccountClosed     = errors.New("account closed")
    ErrDuplicatePayment  = errors.New("duplicate payment detected")
    ErrSelfPayment       = errors.New("cannot pay to self")
)
```

## Error Wrapping

```go
import (
    "errors"
    "fmt"
    "database/sql"
)

// Repository layer: wrap database errors
func (r *PaymentRepository) GetByID(ctx context.Context, id string) (*Payment, error) {
    row := r.db.QueryRowContext(ctx, "SELECT * FROM payments WHERE id = $1", id)

    var p Payment
    if err := row.Scan(&p.ID, &p.Amount, &p.Status); err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            // Wrap with domain context
            return nil, fmt.Errorf("getting payment by id %s: %w", id, model.ErrNotFound)
        }
        return nil, fmt.Errorf("scanning payment row: %w", err)
    }

    return &p, nil
}

// Service layer: add business context
func (s *PaymentService) Transfer(ctx context.Context, cmd TransferCommand) error {
    fromAccount, err := s.accountRepo.GetByID(ctx, cmd.FromAccount)
    if err != nil {
        return fmt.Errorf("fetching source account: %w", err)
    }

    if fromAccount.Balance.LessThan(cmd.Amount) {
        return fmt.Errorf(
            "transferring %s from account %s: %w",
            cmd.Amount, cmd.FromAccount,
            model.ErrInsufficientFunds,
        )
    }

    // ... execute transfer
    return nil
}
```

## Error Checking

```go
func (h *PaymentHandler) CreatePayment(w http.ResponseWriter, r *http.Request) {
    result, err := h.service.CreatePayment(r.Context(), cmd)
    if err != nil {
        // Check specific errors
        switch {
        case errors.Is(err, model.ErrInsufficientFunds):
            respondError(w, http.StatusUnprocessableEntity, "INSUFFICIENT_FUNDS", err.Error())
        case errors.Is(err, model.ErrAccountClosed):
            respondError(w, http.StatusForbidden, "ACCOUNT_CLOSED", err.Error())
        case errors.Is(err, model.ErrNotFound):
            respondError(w, http.StatusNotFound, "NOT_FOUND", err.Error())
        case errors.Is(err, model.ErrDuplicatePayment):
            respondError(w, http.StatusConflict, "DUPLICATE_PAYMENT", err.Error())
        default:
            // Unexpected error - log and return 500
            h.logger.Error("Unexpected error", zap.Error(err))
            respondError(w, http.StatusInternalServerError, "INTERNAL_ERROR", "Internal server error")
        }
        return
    }

    respondJSON(w, http.StatusCreated, result)
}
```

## Custom Error Types

```go
// ValidationError for input validation failures
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error on field '%s': %s", e.Field, e.Message)
}

// Multi-field validation
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

// Usage
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

## Error Checking with Unwrap

```go
// Custom error type with unwrap
type DatabaseError struct {
    Operation string
    Query     string
    Err       error
}

func (e *DatabaseError) Error() string {
    return fmt.Sprintf("database error during %s: %v", e.Operation, e.Err)
}

func (e *DatabaseError) Unwrap() error {
    return e.Err
}

// Usage with errors.Is and errors.As
func handleDBError(err error) {
    var dbErr *DatabaseError
    if errors.As(err, &dbErr) {
        // Access wrapped error's fields
        log.Printf("Operation: %s, Query: %s", dbErr.Operation, dbErr.Query)
    }

    // Check for specific underlying error
    if errors.Is(err, context.DeadlineExceeded) {
        // Database query timed out
        return retryQuery()
    }
}
```

## Banking Error Response

```go
// Standardized error response
type APIError struct {
    Code      string    `json:"code"`
    Message   string    `json:"message"`
    Details   any       `json:"details,omitempty"`
    RequestID string    `json:"request_id"`
    Timestamp time.Time `json:"timestamp"`
}

func respondError(w http.ResponseWriter, statusCode int, code string, message string) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(statusCode)

    err := json.NewEncoder(w).Encode(APIError{
        Code:      code,
        Message:   message,
        RequestID: getRequestID(r.Context()),
        Timestamp: time.Now().UTC(),
    })
    if err != nil {
        log.Printf("Failed to write error response: %v", err)
    }
}

// Example error response
// {
//   "code": "INSUFFICIENT_FUNDS",
//   "message": "Account balance insufficient for requested transfer",
//   "details": {
//     "available_balance": "5000.00",
//     "requested_amount": "7500.00",
//     "currency": "USD"
//   },
//   "request_id": "req-7f3a2b1c",
//   "timestamp": "2026-04-10T08:00:00Z"
// }
```

## Panic Recovery

```go
// internal/middleware/recovery.go
package middleware

import (
    "net/http"
    "runtime/debug"

    "go.uber.org/zap"
)

func Recovery(logger *zap.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            defer func() {
                if rec := recover(); rec != nil {
                    logger.Error("Panic recovered",
                        zap.Any("panic", rec),
                        zap.String("stack", string(debug.Stack())),
                        zap.String("path", r.URL.Path),
                    )
                    respondError(w, http.StatusInternalServerError, "INTERNAL_ERROR", "Internal server error")
                }
            }()
            next.ServeHTTP(w, r)
        })
    }
}
```

## Common Mistakes

1. **Ignoring errors**: `result, _ := doSomething()` - errors silently lost
2. **String comparison for errors**: `if err.Error() == "not found"` instead of `errors.Is`
3. **Not wrapping errors**: Losing context about where and why error occurred
4. **Panicking for expected errors**: Use `return err` not `panic(err)`
5. **No error response standardization**: Different error formats across endpoints
6. **Logging errors without context**: `log.Print(err)` without request ID or path
7. **Revealing internal errors**: Returning database errors to API clients

## Interview Questions

1. How do you distinguish between expected and unexpected errors in a handler?
2. Design an error wrapping strategy for a service that calls 3 repositories.
3. What is the difference between `errors.Is()` and `errors.As()`?
4. How do you handle validation errors that span multiple fields?
5. What happens when you `panic()` in an HTTP handler?

## Cross-References

- `./service-structure.md` - Handler error response patterns
- `./testing.md` - Testing error conditions
- `../api-design/README.md` - Standardized error response format
- `../resilience-patterns/README.md` - Error handling in distributed systems
