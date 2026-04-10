# gRPC Service Design

## Why gRPC

gRPC provides high-performance, type-safe RPC over HTTP/2. In banking microservices, gRPC reduces latency (binary protocol), enforces contracts (protobuf), and enables streaming (real-time data). It is ideal for inter-service communication where both client and server are controlled.

## Protocol Buffer Definition

```protobuf
// api/v1/account.proto
syntax = "proto3";
package api.v1;
option go_package = "github.com/bank/account-service/api/v1;v1";

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

// Account service
service AccountService {
    // Unary RPC
    rpc GetAccount(GetAccountRequest) returns (GetAccountResponse);

    // Server streaming: server sends multiple responses
    rpc GetTransactions(GetTransactionsRequest) returns (stream Transaction);

    // Client streaming: client sends multiple requests
    rpc BulkCreateTransactions(stream CreateTransactionRequest) returns (BatchResult);

    // Bidirectional streaming: both sides stream
    rpc MonitorAccount(MonitorAccountRequest) returns (stream AccountEvent);
}

message GetAccountRequest {
    string account_id = 1;
}

message GetAccountResponse {
    Account account = 1;
}

message Account {
    string id = 1;
    string customer_id = 2;
    string account_number = 3;
    string account_type = 4;
    string currency = 5;
    string balance = 6;  // String for decimal precision
    AccountStatus status = 7;
    google.protobuf.Timestamp created_at = 8;
}

enum AccountStatus {
    ACCOUNT_STATUS_UNSPECIFIED = 0;
    ACCOUNT_STATUS_PENDING = 1;
    ACCOUNT_STATUS_ACTIVE = 2;
    ACCOUNT_STATUS_SUSPENDED = 3;
    ACCOUNT_STATUS_CLOSED = 4;
}

message GetTransactionsRequest {
    string account_id = 1;
    int32 limit = 2;
    string cursor = 3;
}

message Transaction {
    string id = 1;
    string account_id = 2;
    string amount = 3;
    string currency = 4;
    TransactionType type = 5;
    string reference = 6;
    google.protobuf.Timestamp created_at = 7;
}

enum TransactionType {
    TRANSACTION_TYPE_UNSPECIFIED = 0;
    TRANSACTION_TYPE_DEBIT = 1;
    TRANSACTION_TYPE_CREDIT = 2;
}

message CreateTransactionRequest {
    string account_id = 1;
    string amount = 2;
    string currency = 3;
    string reference = 4;
}

message BatchResult {
    int32 total = 1;
    int32 succeeded = 2;
    int32 failed = 3;
    repeated BatchResultItem results = 4;
}

message BatchResultItem {
    int32 index = 1;
    string transaction_id = 2;
    string error = 3;
}

message MonitorAccountRequest {
    string account_id = 1;
}

message AccountEvent {
    string event_id = 1;
    string account_id = 2;
    AccountEventType type = 3;
    string details = 4;
    google.protobuf.Timestamp timestamp = 5;
}

enum AccountEventType {
    ACCOUNT_EVENT_UNSPECIFIED = 0;
    ACCOUNT_EVENT_BALANCE_CHANGED = 1;
    ACCOUNT_EVENT_STATUS_CHANGED = 2;
    ACCOUNT_EVENT_TRANSACTION = 3;
}
```

## Server Implementation

```go
// internal/grpcserver/account.go
package grpcserver

import (
    "context"
    "fmt"

    v1 "github.com/bank/account-service/api/v1"
    "github.com/bank/account-service/internal/model"
    "github.com/bank/account-service/internal/service"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
    "google.golang.org/protobuf/types/known/timestamppb"
)

type AccountServer struct {
    v1.UnimplementedAccountServiceServer
    accountSvc service.AccountService
}

func NewAccountServer(svc service.AccountService) *AccountServer {
    return &AccountServer{accountSvc: svc}
}

func (s *AccountServer) GetAccount(
    ctx context.Context,
    req *v1.GetAccountRequest,
) (*v1.GetAccountResponse, error) {
    if req.GetAccountId() == "" {
        return nil, status.Error(codes.InvalidArgument, "account_id is required")
    }

    account, err := s.accountSvc.GetAccount(ctx, req.GetAccountId())
    if err != nil {
        if service.IsNotFound(err) {
            return nil, status.Errorf(codes.NotFound, "account %s not found", req.GetAccountId())
        }
        return nil, status.Errorf(codes.Internal, "internal error")
    }

    return &v1.GetAccountResponse{
        Account: toProtoAccount(account),
    }, nil
}

// Server streaming: send transactions one at a time
func (s *AccountServer) GetTransactions(
    req *v1.GetTransactionsRequest,
    stream v1.AccountService_GetTransactionsServer,
) error {
    if req.GetAccountId() == "" {
        return status.Error(codes.InvalidArgument, "account_id is required")
    }

    limit := int(req.GetLimit())
    if limit <= 0 {
        limit = 50
    }
    if limit > 100 {
        limit = 100
    }

    transactions, err := s.accountSvc.GetTransactions(
        stream.Context(),
        req.GetAccountId(),
        limit,
        req.GetCursor(),
    )
    if err != nil {
        return status.Errorf(codes.Internal, "failed to fetch transactions")
    }

    for _, txn := range transactions {
        if err := stream.Send(toProtoTransaction(txn)); err != nil {
            return err
        }
    }

    return nil
}

// Client streaming: receive multiple requests, return single result
func (s *AccountServer) BulkCreateTransactions(
    stream v1.AccountService_BulkCreateTransactionsServer,
) error {
    var results []*v1.BatchResultItem
    var succeeded, failed int32

    for {
        req, err := stream.Recv()
        if err != nil {
            break // Stream closed
        }

        result := &v1.BatchResultItem{Index: int32(len(results))}

        _, err = s.accountSvc.CreateTransaction(stream.Context(), model.CreateTransactionCommand{
            AccountID: req.GetAccountId(),
            Amount:    req.GetAmount(),
            Currency:  req.GetCurrency(),
            Reference: req.GetReference(),
        })

        if err != nil {
            result.Error = err.Error()
            failed++
        } else {
            result.TransactionId = "TXN-001" // Actual ID
            succeeded++
        }

        results = append(results, result)
    }

    return stream.SendAndClose(&v1.BatchResult{
        Total:     int32(len(results)),
        Succeeded: succeeded,
        Failed:    failed,
        Results:   results,
    })
}

// Bidirectional streaming: monitor account events in real-time
func (s *AccountServer) MonitorAccount(
    req *v1.MonitorAccountRequest,
    stream v1.AccountService_MonitorAccountServer,
) error {
    // Subscribe to account events
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

// Helper: convert domain model to protobuf
func toProtoAccount(a *model.Account) *v1.Account {
    return &v1.Account{
        Id:            a.ID,
        CustomerId:    a.CustomerID,
        AccountNumber: a.AccountNumber,
        AccountType:   a.AccountType,
        Currency:      a.Currency,
        Balance:       a.Balance.String(),
        Status:        toProtoAccountStatus(a.Status),
        CreatedAt:     timestamppb.New(a.CreatedAt),
    }
}
```

## Server Setup

```go
// cmd/server/main.go
package main

import (
    "net"

    v1 "github.com/bank/account-service/api/v1"
    "github.com/bank/account-service/internal/grpcserver"
    "google.golang.org/grpc"
    "google.golang.org/grpc/keepalive"
    "google.golang.org/grpc/reflection"
)

func main() {
    // Create gRPC server with keepalive
    srv := grpc.NewServer(
        grpc.KeepaliveParams(keepalive.ServerParameters{
            MaxConnectionIdle:     5 * time.Minute,
            MaxConnectionAge:      10 * time.Minute,
            MaxConnectionAgeGrace: 5 * time.Minute,
        }),
        grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
            MinTime:             10 * time.Second,
            PermitWithoutStream: true,
        }),
        grpc.MaxRecvMsgSize(10 * 1024 * 1024),   // 10MB
        grpc.MaxSendMsgSize(10 * 1024 * 1024),   // 10MB
    )

    // Register services
    v1.RegisterAccountServiceServer(srv, accountServer)

    // Enable reflection (for grpcurl, grpcui)
    reflection.Register(srv)

    // Start listening
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }

    log.Println("gRPC server starting on :50051")
    if err := srv.Serve(lis); err != nil {
        log.Fatalf("failed to serve: %v", err)
    }
}
```

## Client Implementation

```go
// Client connecting to gRPC service
func NewAccountClient(addr string) (v1.AccountServiceClient, error) {
    conn, err := grpc.Dial(addr,
        grpc.WithTransportCredentials(insecure.NewCredentials()),  // Use TLS in production
        grpc.WithDefaultCallOptions(
            grpc.MaxCallRecvMsgSize(10*1024*1024),
        ),
    )
    if err != nil {
        return nil, fmt.Errorf("dialing gRPC server: %w", err)
    }

    return v1.NewAccountServiceClient(conn), nil
}

// Usage
client, err := NewAccountClient("localhost:50051")

// Unary call
resp, err := client.GetAccount(ctx, &v1.GetAccountRequest{
    AccountId: "ACC-001",
})

// Server streaming
stream, err := client.GetTransactions(ctx, &v1.GetTransactionsRequest{
    AccountId: "ACC-001",
    Limit:     50,
})
for {
    txn, err := stream.Recv()
    if err == io.EOF {
        break
    }
    if err != nil {
        log.Fatalf("stream error: %v", err)
    }
    fmt.Printf("Transaction: %s\n", txn.Id)
}
```

## Error Handling

```go
// gRPC status codes mapping
func handleError(err error) error {
    switch {
    case errors.Is(err, model.ErrNotFound):
        return status.Errorf(codes.NotFound, "resource not found")
    case errors.Is(err, model.ErrUnauthorized):
        return status.Errorf(codes.Unauthenticated, "unauthenticated")
    case errors.Is(err, model.ErrInvalidInput):
        return status.Errorf(codes.InvalidArgument, "invalid input: %v", err)
    case errors.Is(err, model.ErrDuplicatePayment):
        return status.Errorf(codes.AlreadyExists, "duplicate payment")
    default:
        return status.Errorf(codes.Internal, "internal error")
    }
}

// Client-side error checking
resp, err := client.GetAccount(ctx, req)
if err != nil {
    st, ok := status.FromError(err)
    if ok {
        switch st.Code() {
        case codes.NotFound:
            log.Printf("Account not found: %s", st.Message())
        case codes.Unavailable:
            log.Printf("Service unavailable: %s", st.Message())
        case codes.DeadlineExceeded:
            log.Printf("Request timeout")
        }
    }
}
```

## Interceptors (Middleware)

```go
// Unary interceptor for logging
func LoggingInterceptor() grpc.UnaryServerInterceptor {
    return func(
        ctx context.Context,
        req any,
        info *grpc.UnaryServerInfo,
        handler grpc.UnaryHandler,
    ) (any, error) {
        start := time.Now()
        resp, err := handler(ctx, req)
        duration := time.Since(start)

        log.Printf(
            "gRPC %s - %v - %v",
            info.FullMethod,
            duration,
            err,
        )

        return resp, err
    }
}

// Apply interceptor
srv := grpc.NewServer(
    grpc.UnaryInterceptor(LoggingInterceptor()),
)
```

## Common Mistakes

1. **No message size limits**: Unbounded messages cause memory exhaustion
2. **Ignoring keepalive**: Connections silently drop, client hangs
3. **Not handling stream errors**: Missing `io.EOF` check on stream.Recv()
4. **Using HTTP/1.x load balancer**: gRPC requires HTTP/2
5. **No TLS in production**: Credentials sent in plaintext
6. **No deadline/timeout**: Requests hang indefinitely
7. **Not enabling reflection**: Cannot debug with grpcurl/grpcui

## Interview Questions

1. When would you use gRPC vs REST for inter-service communication?
2. Design a streaming endpoint that monitors account balance changes.
3. How do you handle backward compatibility when adding fields to protobuf?
4. What are the 4 types of gRPC streaming and when to use each?
5. How do you implement authentication in gRPC?

## Cross-References

- `./service-structure.md` - gRPC service in project layout
- `./error-handling.md` - gRPC error patterns
- `../microservices/README.md` - gRPC for inter-service communication
- `../api-design/README.md` - gRPC vs REST comparison
- `../concurrency/README.md` - Streaming and concurrency patterns
