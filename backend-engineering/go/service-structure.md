# Production Go Service Structure

## Service Layout

```
banking-payment-service/
├── cmd/
│   └── server/
│       └── main.go              # Entry point, wiring, server startup
├── internal/                    # Private application code
│   ├── config/
│   │   └── config.go            # Configuration loading and validation
│   ├── handler/                 # HTTP/REST handlers
│   │   ├── handler.go           # Handler interface and base
│   │   ├── payment.go           # Payment endpoints
│   │   ├── account.go           # Account endpoints
│   │   └── health.go            # Health check endpoints
│   ├── service/                 # Business logic
│   │   ├── payment.go           # Payment business logic
│   │   ├── account.go           # Account business logic
│   │   └── fraud.go             # Fraud detection logic
│   ├── repository/              # Data access layer
│   │   ├── payment.go           # Payment database operations
│   │   ├── account.go           # Account database operations
│   │   └── cache.go             # Redis cache operations
│   ├── model/                   # Domain models
│   │   ├── payment.go           # Payment domain model
│   │   ├── account.go           # Account domain model
│   │   └── money.go             # Money/Currency types
│   ├── middleware/              # HTTP middleware
│   │   ├── auth.go              # Authentication middleware
│   │   ├── logging.go           # Request logging
│   │   ├── recovery.go          # Panic recovery
│   │   └── cors.go              # CORS handling
│   └── server/                  # Server setup
│       └── server.go            # HTTP server configuration
├── pkg/                         # Public packages (reusable)
│   └── validator/
│       ├── iban.go              # IBAN validation
│       └── swift.go             # SWIFT code validation
├── api/                         # API definitions
│   └── openapi/
│       └── v1.yaml              # OpenAPI specification
├── deployments/
│   ├── Dockerfile               # Production Docker image
│   └── k8s/
│       ├── deployment.yaml      # Kubernetes deployment
│       └── service.yaml         # Kubernetes service
├── testutil/                    # Test utilities
│   └── fixtures.go              # Test data generators
├── go.mod
├── go.sum
├── Makefile
└── README.md
```

## Entry Point

```go
// cmd/server/main.go
package main

import (
    "context"
    "fmt"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/bank/payment-service/internal/config"
    "github.com/bank/payment-service/internal/handler"
    "github.com/bank/payment-service/internal/middleware"
    "github.com/bank/payment-service/internal/repository"
    "github.com/bank/payment-service/internal/server"
    "github.com/bank/payment-service/internal/service"
    "go.uber.org/zap"
)

func main() {
    // Initialize logger
    logger, err := zap.NewProduction()
    if err != nil {
        fmt.Fprintf(os.Stderr, "Failed to initialize logger: %v\n", err)
        os.Exit(1)
    }
    defer logger.Sync()

    // Load configuration
    cfg, err := config.Load()
    if err != nil {
        logger.Fatal("Failed to load config", zap.Error(err))
    }

    // Initialize database
    db, err := repository.NewDatabase(cfg.DatabaseURL)
    if err != nil {
        logger.Fatal("Failed to connect to database", zap.Error(err))
    }
    defer db.Close()

    // Initialize Redis
    rdb, err := repository.NewRedis(cfg.RedisURL)
    if err != nil {
        logger.Fatal("Failed to connect to Redis", zap.Error(err))
    }

    // Initialize services
    paymentRepo := repository.NewPaymentRepository(db)
    accountRepo := repository.NewAccountRepository(db)
    paymentSvc := service.NewPaymentService(paymentRepo, accountRepo, logger)

    // Initialize handlers
    paymentHandler := handler.NewPaymentHandler(paymentSvc, logger)
    healthHandler := handler.NewHealthHandler(db, rdb, logger)

    // Set up router
    mux := http.NewServeMux()

    // Middleware chain
    chain := middleware.NewChain(
        middleware.RequestID(),
        middleware.Logging(logger),
        middleware.Recovery(logger),
        middleware.Cors(cfg.AllowedOrigins),
    )

    // Register routes
    mux.Handle("POST /v1/payments", chain.Then(paymentHandler.CreatePayment))
    mux.Handle("GET /v1/payments/{id}", chain.Then(paymentHandler.GetPayment))
    mux.Handle("GET /v1/accounts/{id}/transactions", chain.Then(paymentHandler.GetTransactions))
    mux.Handle("GET /health", healthHandler.Readiness)
    mux.Handle("GET /health/live", healthHandler.Liveness)

    // Create and start server
    srv := server.New(mux, server.Options{
        Addr:         fmt.Sprintf(":%d", cfg.Port),
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 30 * time.Second,
        IdleTimeout:  120 * time.Second,
    })

    // Graceful shutdown
    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()

    go func() {
        logger.Info("Starting server", zap.Int("port", cfg.Port))
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            logger.Fatal("Server failed", zap.Error(err))
        }
    }()

    // Wait for shutdown signal
    <-ctx.Done()
    logger.Info("Shutting down server...")

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(shutdownCtx); err != nil {
        logger.Fatal("Server shutdown failed", zap.Error(err))
    }

    logger.Info("Server stopped gracefully")
}
```

## Configuration

```go
// internal/config/config.go
package config

import (
    "fmt"
    "os"
    "time"

    "github.com/kelseyhightower/envconfig"
)

type Config struct {
    Port           int           `envconfig:"PORT" default:"8080"`
    Environment    string        `envconfig:"ENVIRONMENT" default:"development"`
    DatabaseURL    string        `envconfig:"DATABASE_URL" required:"true"`
    DatabasePool   int           `envconfig:"DATABASE_POOL" default:"20"`
    RedisURL       string        `envconfig:"REDIS_URL" default:"redis://localhost:6379"`
    AllowedOrigins []string      `envconfig:"ALLOWED_ORIGINS" default:"*"`
    JWTSecret      string        `envconfig:"JWT_SECRET" required:"true"`
    RequestTimeout time.Duration `envconfig:"REQUEST_TIMEOUT" default:"30s"`
}

func Load() (*Config, error) {
    var cfg Config
    if err := envconfig.Process("", &cfg); err != nil {
        return nil, fmt.Errorf("processing config: %w", err)
    }

    if err := cfg.Validate(); err != nil {
        return nil, fmt.Errorf("validating config: %w", err)
    }

    return &cfg, nil
}

func (c *Config) Validate() error {
    if c.DatabaseURL == "" {
        return fmt.Errorf("DATABASE_URL is required")
    }
    if len(c.JWTSecret) < 32 {
        return fmt.Errorf("JWT_SECRET must be at least 32 characters")
    }
    if c.Port < 1 || c.Port > 65535 {
        return fmt.Errorf("PORT must be between 1 and 65535")
    }
    return nil
}
```

## Handler Pattern

```go
// internal/handler/payment.go
package handler

import (
    "encoding/json"
    "net/http"

    "github.com/bank/payment-service/internal/model"
    "github.com/bank/payment-service/internal/service"
    "go.uber.org/zap"
)

type PaymentHandler struct {
    service service.PaymentService
    logger  *zap.Logger
}

func NewPaymentHandler(svc service.PaymentService, logger *zap.Logger) *PaymentHandler {
    return &PaymentHandler{service: svc, logger: logger}
}

func (h *PaymentHandler) CreatePayment(w http.ResponseWriter, r *http.Request) {
    var req CreatePaymentRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "INVALID_REQUEST", "Invalid JSON body")
        return
    }

    // Validate request
    if err := req.Validate(); err != nil {
        respondError(w, http.StatusBadRequest, "VALIDATION_ERROR", err.Error())
        return
    }

    // Execute payment
    result, err := h.service.CreatePayment(r.Context(), model.CreatePaymentCommand{
        FromAccount: req.FromAccount,
        ToAccount:   req.ToAccount,
        Amount:      req.Amount,
        Currency:    req.Currency,
        Reference:   req.Reference,
    })
    if err != nil {
        h.logger.Error("Failed to create payment", zap.Error(err))
        switch {
        case service.IsInsufficientFunds(err):
            respondError(w, http.StatusUnprocessableEntity, "INSUFFICIENT_FUNDS", err.Error())
        case service.IsAccountNotFound(err):
            respondError(w, http.StatusNotFound, "ACCOUNT_NOT_FOUND", err.Error())
        default:
            respondError(w, http.StatusInternalServerError, "INTERNAL_ERROR", "An unexpected error occurred")
        }
        return
    }

    respondJSON(w, http.StatusCreated, result)
}
```

## Server Configuration

```go
// internal/server/server.go
package server

import (
    "context"
    "net/http"
    "time"
)

type Options struct {
    Addr              string
    ReadTimeout       time.Duration
    WriteTimeout      time.Duration
    IdleTimeout       time.Duration
    ReadHeaderTimeout time.Duration
}

func New(handler http.Handler, opts Options) *http.Server {
    return &http.Server{
        Addr:              opts.Addr,
        Handler:           handler,
        ReadTimeout:       opts.ReadTimeout,
        WriteTimeout:      opts.WriteTimeout,
        IdleTimeout:       opts.IdleTimeout,
        ReadHeaderTimeout: 5 * time.Second,
    }
}
```

## Database Connection

```go
// internal/repository/database.go
package repository

import (
    "database/sql"
    "fmt"
    "time"

    _ "github.com/jackc/pgx/v5/stdlib"
)

func NewDatabase(dsn string) (*sql.DB, error) {
    db, err := sql.Open("pgx", dsn)
    if err != nil {
        return nil, fmt.Errorf("opening database: %w", err)
    }

    // Connection pool settings
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(10)
    db.SetConnMaxLifetime(30 * time.Minute)
    db.SetConnMaxIdleTime(5 * time.Minute)

    // Test connection
    if err := db.Ping(); err != nil {
        return nil, fmt.Errorf("pinging database: %w", err)
    }

    return db, nil
}
```

## Docker Deployment

```dockerfile
# deployments/Dockerfile
FROM golang:1.22-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /payment-service ./cmd/server

# Production image
FROM alpine:3.19

RUN apk --no-cache add ca-certificates tzdata

WORKDIR /app
COPY --from=builder /payment-service .

# Non-root user
RUN adduser -D appuser
USER appuser

EXPOSE 8080

ENTRYPOINT ["./payment-service"]
```

## Common Mistakes

1. **Putting everything in main.go**: No separation of concerns
2. **Using `internal/` for public packages**: `internal/` cannot be imported by other modules
3. **No graceful shutdown**: Active requests terminated during deployment
4. **Hardcoded configuration**: Not using environment variables
5. **No health checks**: Kubernetes cannot determine pod readiness
6. **Ignoring connection pool limits**: Unlimited connections exhaust database
7. **Not using structured logging**: `fmt.Println` instead of `zap.Logger`

## Interview Questions

1. How do you structure a Go service with 20+ endpoints?
2. Design a handler that validates requests, calls a service, and returns errors.
3. How do you implement graceful shutdown in Go?
4. What is the purpose of the `internal/` directory?
5. How do you manage configuration across environments (dev, staging, prod)?

## Cross-References

- `./error-handling.md` - Error wrapping and sentinel errors
- `./concurrency.md` - Goroutines and channels in handlers
- `./testing.md` - Testing Go handlers and services
- `../api-design/README.md` - REST API design for Go services
- `../microservices/README.md` - Go in microservices architecture
