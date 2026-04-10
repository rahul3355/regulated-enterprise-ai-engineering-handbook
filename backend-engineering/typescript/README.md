# TypeScript for Banking Backend

## Why TypeScript for Backend

TypeScript adds compile-time type safety to Node.js. For banking GenAI platforms, TypeScript powers API gateways, BFF (Backend-for-Frontend) layers, webhook processors, and real-time notification services. The shared type system between frontend and backend reduces integration bugs.

## Strengths

### Type Safety

```typescript
// Compile-time type checking catches bugs before runtime
interface Payment {
    id: string;
    fromAccount: string;
    toAccount: string;
    amount: Decimal;
    currency: Currency;
    status: PaymentStatus;
    createdAt: Date;
}

// TypeScript catches mismatched fields at compile time
const payment: Payment = {
    id: 'PAY-001',
    fromAccount: 'ACC-001',
    amount: '100',  // Error: Type 'string' is not assignable to type 'Decimal'
};
```

### Async-First

```typescript
// Node.js is async by design - no GIL, no thread pools needed
async function processPayments(payments: Payment[]): Promise<PaymentResult[]> {
    // All I/O is non-blocking
    const results = await Promise.all(
        payments.map(p => executePayment(p))
    );
    return results;
}
```

### Shared Types with Frontend

```typescript
// Shared types between frontend and backend
// Single source of truth for API contracts
export interface PaymentRequest {
    fromAccount: string;
    toAccount: string;
    amount: string;
    currency: string;
    reference?: string;
}

export interface PaymentResponse {
    id: string;
    status: PaymentStatus;
    amount: string;
    createdAt: string;
}

// Frontend imports same types:
// import { PaymentRequest, PaymentResponse } from '@bank/shared-types';
```

## Weaknesses

### Single-Threaded

```typescript
// CPU-bound work blocks the event loop
function computeRiskScore(data: any): number {
    // Heavy computation blocks all other requests!
    return heavyMLInference(data);
}

// Solution: Offload to worker threads or separate service
import { Worker } from 'worker_threads';

async function computeRiskScoreAsync(data: any): Promise<number> {
    return new Promise((resolve, reject) => {
        const worker = new Worker('./risk-worker.js', { workerData: data });
        worker.on('message', resolve);
        worker.on('error', reject);
    });
}
```

### Ecosystem Maturity

```
// Fewer enterprise frameworks compared to Java/Spring
// Less mature ORM options (Prisma, TypeORM vs Hibernate)
// Smaller talent pool for senior TypeScript backend engineers
```

## Typical Banking Use Cases

### API Gateway

```typescript
// Gateway routes requests to backend services
import express from 'express';
import { createProxyMiddleware } from 'http-proxy-middleware';

const app = express();

// Authentication middleware
app.use(authMiddleware);

// Route to backend services
app.use('/v1/payments', createProxyMiddleware({
    target: process.env.PAYMENT_SERVICE_URL,
    changeOrigin: true,
    pathRewrite: { '^/v1/payments': '/api/payments' },
}));

app.use('/v1/accounts', createProxyMiddleware({
    target: process.env.ACCOUNT_SERVICE_URL,
    changeOrigin: true,
}));

app.use('/v1/ai', createProxyMiddleware({
    target: process.env.AI_SERVICE_URL,
    changeOrigin: true,
}));
```

### Webhook Processor

```typescript
// Process incoming webhooks from external services
@Controller('webhooks')
export class WebhookController {
    constructor(
        private readonly paymentService: PaymentService,
        private readonly webhookValidator: WebhookValidator,
    ) {}

    @Post('payment-status')
    async handlePaymentStatus(@Body() payload: PaymentWebhookPayload) {
        // Verify webhook signature
        this.webhookValidator.verifySignature(payload);

        // Update local payment status
        await this.paymentService.updateStatus(
            payload.paymentId,
            payload.status,
        );

        // Send notification to customer
        if (payload.status === 'completed') {
            await this.notificationService.sendPaymentConfirmation(
                payload.accountId,
                payload.paymentId,
            );
        }
    }
}
```

### Real-Time Notifications

```typescript
// WebSocket server for real-time payment updates
import { WebSocketServer, WebSocket } from 'ws';

const wss = new WebSocketServer({ port: 8081 });

const clients = new Map<string, Set<WebSocket>>();

wss.on('connection', (ws, req) => {
    const userId = extractUserId(req);

    if (!clients.has(userId)) {
        clients.set(userId, new Set());
    }
    clients.get(userId)!.add(ws);

    ws.on('close', () => {
        clients.get(userId)?.delete(ws);
    });
});

// Broadcast payment update to user
function broadcastPaymentUpdate(userId: string, payment: Payment) {
    const userClients = clients.get(userId);
    if (userClients) {
        const message = JSON.stringify({
            type: 'payment_update',
            payment,
        });
        userClients.forEach(client => {
            if (client.readyState === WebSocket.OPEN) {
                client.send(message);
            }
        });
    }
}
```

## Production Service Layout

```
banking-api-gateway/
├── src/
│   ├── index.ts                 # Application entry point
│   ├── config/
│   │   └── index.ts             # Configuration management
│   ├── middleware/
│   │   ├── auth.ts              # Authentication
│   │   ├── logging.ts           # Request logging
│   │   ├── errorHandler.ts      # Error handling
│   │   └── rateLimiter.ts       # Rate limiting
│   ├── routes/
│   │   ├── payments.ts          # Payment routes
│   │   ├── accounts.ts          # Account routes
│   │   └── health.ts            # Health check routes
│   ├── services/
│   │   ├── paymentService.ts    # Payment business logic
│   │   └── proxyService.ts      # Service proxy
│   ├── types/
│   │   ├── payment.ts           # Payment types
│   │   └── api.ts               # API types
│   └── utils/
│       ├── logger.ts            # Structured logging
│       └── errors.ts            # Custom error classes
├── test/
│   ├── routes/
│   └── services/
├── tsconfig.json
├── package.json
└── Dockerfile
```

## Common Mistakes

1. **Blocking the event loop**: CPU-bound work in request handlers
2. **No type sharing**: Different types between frontend and backend
3. **Not using strict mode**: `strict: true` in tsconfig catches bugs
4. **Unhandled promise rejections**: Missing `.catch()` on async operations
5. **No connection pooling**: Creating new HTTP client per request
6. **Ignoring backpressure**: Streaming large responses without flow control

## Interview Questions

1. How do you handle CPU-intensive work in a Node.js backend service?
2. Design a TypeScript API gateway that routes to 5 backend services.
3. How do you share types between frontend and backend in a monorepo?
4. What happens when an unhandled promise rejection occurs?
5. How do you implement rate limiting in a Node.js service?

## Cross-References

- `./express-nestjs.md` - NestJS production service
- `./type-safety.md` - Advanced TypeScript patterns
- `./testing.md` - Jest and integration testing
- `./performance.md` - Node.js performance tuning
- `../api-design/README.md` - REST API design
