# Advanced TypeScript Patterns for APIs

## Type-Safe API Contracts

### Shared API Types

```typescript
// types/api.ts - Single source of truth for API contracts

// Request types
export interface PaymentRequest {
    fromAccount: string;
    toAccount: string;
    amount: string;  // String for Decimal precision
    currency: Currency;
    reference?: string;
}

export interface CreateAccountRequest {
    customerId: string;
    accountType: AccountType;
    currency: Currency;
    initialDeposit?: string;
}

// Response types
export interface PaymentResponse {
    id: string;
    status: PaymentStatus;
    amount: string;
    currency: Currency;
    createdAt: string;
    executedAt?: string;
}

export interface PaginatedResponse<T> {
    items: T[];
    pagination: {
        page: number;
        limit: number;
        total: number;
        totalPages: number;
    };
}

// Error types
export interface APIError {
    error: {
        code: string;
        message: string;
        details?: Record<string, unknown>;
        requestId?: string;
        timestamp: string;
    };
}
```

### Type-Safe Route Handlers

```typescript
// Using TypeScript to enforce request/response types
type RouteHandler<ReqBody, ReqParams, ResBody> = (
    req: { body: ReqBody; params: ReqParams },
    res: { status(code: number): { json(body: ResBody): void } },
) => Promise<void>;

const createPayment: RouteHandler<
    PaymentRequest,
    {},
    PaymentResponse
> = async (req, res) => {
    const payment = await paymentService.create(req.body);
    res.status(201).json(PaymentResponse.from(payment));
};
```

### Discriminated Unions

```typescript
// Type-safe payment method handling
interface WireTransfer {
    type: 'wire';
    swiftCode: string;
    beneficiaryName: string;
    beneficiaryAccount: string;
}

interface SEPATransfer {
    type: 'sepa';
    iban: string;
    beneficiaryName: string;
    bic?: string;
}

interface DomesticTransfer {
    type: 'domestic';
    accountNumber: string;
    routingNumber: string;
    beneficiaryName: string;
}

// Discriminated union
type PaymentMethod = WireTransfer | SEPATransfer | DomesticTransfer;

// Type-safe processing
function processPayment(method: PaymentMethod): Promise<PaymentResult> {
    switch (method.type) {
        case 'wire':
            // TypeScript knows method is WireTransfer here
            return processWire(method.swiftCode, method.beneficiaryName);
        case 'sepa':
            // TypeScript knows method is SEPATransfer here
            return processSEPA(method.iban, method.beneficiaryName);
        case 'domestic':
            // TypeScript knows method is DomesticTransfer here
            return processDomestic(method.accountNumber, method.routingNumber);
    }
}
```

### Branded Types for Domain Safety

```typescript
// Prevent mixing up different string types
declare const __brand: unique symbol;
type Brand<B> = { [__brand]: B };
type Branded<T, B> = T & Brand<B>;

// Domain types
type AccountId = Branded<string, 'AccountId'>;
type PaymentId = Branded<string, 'PaymentId'>;
type CustomerId = Branded<string, 'CustomerId'>;

// Type-safe functions
function getAccount(id: AccountId): Promise<Account> { ... }
function getPayment(id: PaymentId): Promise<Payment> { ... }

// Usage - TypeScript prevents mixing IDs
const accountId = 'ACC-001' as AccountId;
const paymentId = 'PAY-001' as PaymentId;

getAccount(accountId);    // OK
getAccount(paymentId);    // Error: PaymentId not assignable to AccountId
```

### Template Literal Types

```typescript
// Type-safe event naming
type EventDomain = 'payment' | 'account' | 'customer' | 'compliance';
type EventAction = 'created' | 'updated' | 'deleted' | 'flagged';
type EventName = `${EventDomain}.${EventAction}`;

const event: EventName = 'payment.created';    // OK
const badEvent: EventName = 'payment.expired';  // Error

// Type-safe API paths
type APIVersion = 'v1' | 'v2';
type Entity = 'payments' | 'accounts' | 'customers';
type Action = 'list' | 'create' | 'get' | 'update' | 'delete';
type APIPath = `/api/${APIVersion}/${Entity}` | `/api/${APIVersion}/${Entity}/${string}`;

const path: APIPath = '/api/v1/payments';  // OK
const badPath: APIPath = '/api/v3/payments';  // Error
```

## Generic Validation

```typescript
// Generic validation function
type Validator<T> = (value: unknown) => value is T;

function validate<T>(value: unknown, validator: Validator<T>, typeName: string): T {
    if (!validator(value)) {
        throw new ValidationError(`Expected ${typeName}, got ${typeof value}`);
    }
    return value;
}

// Usage
function isPaymentRequest(value: unknown): value is PaymentRequest {
    return (
        typeof value === 'object' &&
        value !== null &&
        'fromAccount' in value &&
        'toAccount' in value &&
        'amount' in value
    );
}

const payment = validate(req.body, isPaymentRequest, 'PaymentRequest');
// payment is typed as PaymentRequest
```

## Type-Safe Database Queries

```typescript
// Type-safe query builder
interface QueryBuilder<T> {
    where<K extends keyof T>(field: K, value: T[K]): QueryBuilder<T>;
    orderBy<K extends keyof T>(field: K, direction: 'ASC' | 'DESC'): QueryBuilder<T>;
    limit(n: number): QueryBuilder<T>;
    execute(): Promise<T[]>;
}

// Usage - TypeScript ensures field/value type matching
const payments = await queryBuilder<Payment>()
    .where('status', 'COMPLETED')    // OK: status is PaymentStatus
    .where('amount', 100)            // OK: amount is number
    .where('amount', '100')          // Error: string not assignable to number
    .orderBy('createdAt', 'DESC')
    .limit(50)
    .execute();
```

## Runtime Type Checking

```typescript
// Using zod for runtime validation with type inference
import { z } from 'zod';

const PaymentRequestSchema = z.object({
    fromAccount: z.string().min(10).max(34),
    toAccount: z.string().min(10).max(34),
    amount: z.string().regex(/^\d+(\.\d{1,2})?$/),
    currency: z.string().regex(/^[A-Z]{3}$/),
    reference: z.string().max(140).optional(),
});

// Infer TypeScript from schema
type PaymentRequest = z.infer<typeof PaymentRequestSchema>;

// Validate at runtime
function validatePaymentRequest(data: unknown): PaymentRequest {
    return PaymentRequestSchema.parse(data);  // Throws on invalid
}

// Or return result instead of throwing
const result = PaymentRequestSchema.safeParse(data);
if (!result.success) {
    return { error: result.error.format() };
}
const payment = result.data;  // Typed as PaymentRequest
```

## Common Mistakes

1. **Using `any`**: Loses all type safety, defeats TypeScript's purpose
2. **Not using `strict: true`**: Misses null checks, implicit any, and more
3. **No branded types**: Mixing up different string types (AccountId vs PaymentId)
4. **Runtime/type mismatch**: TypeScript types don't match actual runtime validation
5. **Over-engineering types**: Complex type gymnastics for simple cases
6. **Ignoring discriminated unions**: Manual type checking instead of type-safe switches

## Interview Questions

1. How do you prevent mixing up AccountId and PaymentId at compile time?
2. Design a type-safe API client that enforces request/response types.
3. What are discriminated unions and how do they improve type safety?
4. How do you validate runtime data matches TypeScript types?
5. Design template literal types for event naming conventions.

## Cross-References

- `./express-nestjs.md` - Using TypeScript in NestJS
- `./testing.md` - Testing type contracts
- `../api-design/README.md` - API type contracts
- `../backend-testing/README.md` - Contract testing
