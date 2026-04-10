# TypeScript Testing Patterns

## Jest Unit Testing

### Basic Tests

```typescript
// services/payment.service.test.ts
import { PaymentService } from './payment.service';
import { PaymentRepository } from '../repositories/payment.repository';
import { InsufficientFundsError } from '../errors';

describe('PaymentService', () => {
    let paymentService: PaymentService;
    let mockRepo: jest.Mocked<PaymentRepository>;

    beforeEach(() => {
        mockRepo = {
            findById: jest.fn(),
            create: jest.fn(),
            update: jest.fn(),
        } as any;

        paymentService = new PaymentService(mockRepo);
    });

    it('should create payment successfully', async () => {
        // Arrange
        mockRepo.findById
            .mockResolvedValueOnce({ id: 'ACC-001', balance: 1000 })
            .mockResolvedValueOnce({ id: 'ACC-002', balance: 500 });

        mockRepo.create.mockResolvedValue({
            id: 'PAY-001',
            status: 'PENDING',
            amount: 100,
        });

        // Act
        const result = await paymentService.create({
            fromAccount: 'ACC-001',
            toAccount: 'ACC-002',
            amount: 100,
            currency: 'USD',
        });

        // Assert
        expect(result.status).toBe('PENDING');
        expect(result.amount).toBe(100);
        expect(mockRepo.create).toHaveBeenCalledWith(expect.objectContaining({
            fromAccount: 'ACC-001',
            toAccount: 'ACC-002',
        }));
    });

    it('should throw InsufficientFundsError when balance is too low', async () => {
        // Arrange
        mockRepo.findById
            .mockResolvedValueOnce({ id: 'ACC-001', balance: 50 })
            .mockResolvedValueOnce({ id: 'ACC-002', balance: 500 });

        // Act & Assert
        await expect(paymentService.create({
            fromAccount: 'ACC-001',
            toAccount: 'ACC-002',
            amount: 100,
            currency: 'USD',
        })).rejects.toThrow(InsufficientFundsError);
    });
});
```

### Parameterized Tests

```typescript
describe('Payment amount validation', () => {
    const invalidAmounts = [0, -1, -0.01, 1000001];

    test.each(invalidAmounts)('rejects amount %d', async (amount) => {
        await expect(paymentService.create({
            fromAccount: 'ACC-001',
            toAccount: 'ACC-002',
            amount,
            currency: 'USD',
        })).rejects.toThrow('Invalid amount');
    });

    const validCurrencies = ['USD', 'EUR', 'GBP', 'JPY'];

    test.each(validCurrencies)('accepts currency %s', async (currency) => {
        mockRepo.findById.mockResolvedValue({ id: 'ACC-001', balance: 1000 });
        mockRepo.create.mockResolvedValue({ id: 'PAY-001', status: 'PENDING' });

        const result = await paymentService.create({
            fromAccount: 'ACC-001',
            toAccount: 'ACC-002',
            amount: 100,
            currency,
        });

        expect(result.status).toBe('PENDING');
    });
});
```

## Supertest Integration Testing

### API Endpoint Testing

```typescript
// routes/payments.test.ts
import request from 'supertest';
import { INestApplication } from '@nestjs/common';
import { Test, TestingModule } from '@nestjs/testing';
import { PaymentsModule } from '../modules/payments/payments.module';
import { getRepositoryToken } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Payment } from '../modules/payments/entities/payment.entity';

describe('PaymentsController (e2e)', () => {
    let app: INestApplication;
    let paymentRepo: Repository<Payment>;

    beforeAll(async () => {
        const moduleFixture: TestingModule = await Test.createTestingModule({
            imports: [PaymentsModule],
        })
        .overrideProvider(getRepositoryToken(Payment))
        .useValue({
            create: jest.fn(),
            save: jest.fn(),
            findOne: jest.fn(),
            findAndCount: jest.fn(),
        })
        .compile();

        app = moduleFixture.createNestApplication();
        await app.init();

        paymentRepo = moduleFixture.get<Repository<Payment>>(getRepositoryToken(Payment));
    });

    afterAll(async () => {
        await app.close();
    });

    it('POST /api/payments - creates payment', async () => {
        (paymentRepo.save as jest.Mock).mockResolvedValue({
            id: 'PAY-001',
            fromAccount: 'ACC-001',
            toAccount: 'ACC-002',
            amount: 100,
            currency: 'USD',
            status: 'PENDING',
            createdAt: new Date(),
        });

        const response = await request(app.getHttpServer())
            .post('/api/payments')
            .send({
                fromAccount: 'ACC-001',
                toAccount: 'ACC-002',
                amount: 100,
                currency: 'USD',
            })
            .expect(201);

        expect(response.body).toMatchObject({
            id: 'PAY-001',
            status: 'PENDING',
            amount: 100,
            currency: 'USD',
        });
    });

    it('POST /api/payments - validation error', async () => {
        const response = await request(app.getHttpServer())
            .post('/api/payments')
            .send({
                fromAccount: '',  // Invalid
                amount: -100,     // Invalid
            })
            .expect(400);

        expect(response.body.error).toBeDefined();
        expect(response.body.error.code).toBe('VALIDATION_ERROR');
    });

    it('GET /api/payments/:id - not found', async () => {
        (paymentRepo.findOne as jest.Mock).mockResolvedValue(null);

        await request(app.getHttpServer())
            .get('/api/payments/NONEXISTENT')
            .expect(404);
    });
});
```

## Mocking External Services

```typescript
// services/external-gateway.test.ts
import nock from 'nock';
import { ExternalPaymentGateway } from './external-gateway';

describe('ExternalPaymentGateway', () => {
    let gateway: ExternalPaymentGateway;

    beforeEach(() => {
        gateway = new ExternalPaymentGateway({
            baseUrl: 'https://api.external-gateway.com',
            apiKey: 'test-key',
        });

        nock.disableNetConnect();
    });

    afterEach(() => {
        nock.cleanAll();
        nock.enableNetConnect();
    });

    it('initiates payment successfully', async () => {
        nock('https://api.external-gateway.com')
            .post('/v1/payments')
            .reply(200, {
                payment_id: 'EXT-PAY-001',
                status: 'accepted',
            });

        const result = await gateway.initiatePayment({
            amount: 100,
            currency: 'USD',
            recipient: 'ACC-002',
        });

        expect(result.paymentId).toBe('EXT-PAY-001');
        expect(result.status).toBe('accepted');
    });

    it('handles gateway timeout', async () => {
        nock('https://api.external-gateway.com')
            .post('/v1/payments')
            .socketDelay(5000);

        await expect(gateway.initiatePayment({
            amount: 100,
            currency: 'USD',
            recipient: 'ACC-002',
        })).rejects.toThrow('Connection timeout');
    });
});
```

## Test Configuration

```typescript
// jest.config.js
module.exports = {
    preset: 'ts-jest',
    testEnvironment: 'node',
    roots: ['<rootDir>/src', '<rootDir>/test'],
    testMatch: ['**/*.test.ts', '**/*.spec.ts'],
    transform: {
        '^.+\\.ts$': 'ts-jest',
    },
    collectCoverageFrom: [
        'src/**/*.ts',
        '!src/**/*.d.ts',
        '!src/**/*.module.ts',
        '!src/main.ts',
    ],
    coverageThreshold: {
        global: {
            branches: 80,
            functions: 90,
            lines: 90,
            statements: 90,
        },
    },
    coverageReporters: ['text', 'lcov', 'html'],
    setupFilesAfterEnv: ['<rootDir>/test/setup.ts'],
};
```

## Common Mistakes

1. **Not mocking dependencies**: Tests hit real database/external APIs
2. **Testing implementation**: Tests break on refactoring
3. **No assertion on mock calls**: Not verifying interactions
4. **Shared state between tests**: Tests interfere with each other
5. **Using `any` in mocks**: Loses type safety in tests
6. **Slow tests**: No isolation, hitting real external services

## Interview Questions

1. How do you test a NestJS controller that depends on a database?
2. Design integration tests for a payment API with mocked external gateway.
3. How do you parameterize tests for 20 validation cases?
4. What is the difference between unit and e2e tests in NestJS?
5. How do you test async code that streams data?

## Cross-References

- `./express-nestjs.md` - NestJS service testing
- `./type-safety.md` - Testing type contracts
- `../backend-testing/README.md` - Overall testing strategy
- `../api-design/README.md` - Testing API contracts
