# NestJS Production Service Structure

## Why NestJS

NestJS is the enterprise TypeScript framework inspired by Spring and Angular. It provides dependency injection, modular architecture, decorators, and built-in testing utilities. For banking GenAI platforms, NestJS brings Java-like structure to Node.js.

## Service Layout

```
banking-nestjs-service/
├── src/
│   ├── main.ts                    # Application bootstrap
│   ├── app.module.ts              # Root module
│   ├── config/
│   │   ├── config.module.ts
│   │   └── app.config.ts          # Configuration provider
│   ├── common/
│   │   ├── filters/
│   │   │   └── global-exception.filter.ts
│   │   ├── interceptors/
│   │   │   ├── logging.interceptor.ts
│   │   │   └── timeout.interceptor.ts
│   │   ├── guards/
│   │   │   └── jwt-auth.guard.ts
│   │   ├── decorators/
│   │   │   └── current-user.decorator.ts
│   │   └── dtos/
│   │       ├── pagination.dto.ts
│   │       └── error-response.dto.ts
│   ├── modules/
│   │   ├── payments/
│   │   │   ├── payments.module.ts
│   │   │   ├── payments.controller.ts
│   │   │   ├── payments.service.ts
│   │   │   ├── payments.repository.ts
│   │   │   ├── dto/
│   │   │   │   ├── create-payment.dto.ts
│   │   │   │   └── payment-response.dto.ts
│   │   │   ├── entities/
│   │   │   │   └── payment.entity.ts
│   │   │   └── payments.controller.spec.ts
│   │   ├── accounts/
│   │   │   ├── accounts.module.ts
│   │   │   ├── accounts.controller.ts
│   │   │   └── accounts.service.ts
│   │   └── ai/
│   │       ├── ai.module.ts
│   │       ├── ai.controller.ts
│   │       └── ai.service.ts
│   └── database/
│       ├── database.module.ts
│       └── database.providers.ts
├── test/
│   ├── jest-e2e.json
│   └── app.e2e-spec.ts
├── tsconfig.json
├── tsconfig.build.json
├── nest-cli.json
├── package.json
└── Dockerfile
```

## Application Bootstrap

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { ValidationPipe, Logger } from '@nestjs/common';
import { AppModule } from './app.module';
import { GlobalExceptionFilter } from './common/filters/global-exception.filter';
import { LoggingInterceptor } from './common/interceptors/logging.interceptor';
import { TimeoutInterceptor } from './common/interceptors/timeout.interceptor';
import { ConfigService } from '@nestjs/config';

async function bootstrap() {
    const logger = new Logger('Bootstrap');

    const app = await NestFactory.create(AppModule, {
        logger: ['error', 'warn', 'log', 'debug', 'verbose'],
    });

    // Global prefix
    app.setGlobalPrefix('api');

    // Global validation pipe
    app.useGlobalPipes(
        new ValidationPipe({
            whitelist: true,           // Strip unknown properties
            forbidNonWhitelisted: true, // Reject unknown properties
            transform: true,           // Transform payloads to DTOs
            transformOptions: {
                enableImplicitConversion: true,
            },
            disableErrorMessages: process.env.NODE_ENV === 'production',
        }),
    );

    // Global exception filter
    app.useGlobalFilters(new GlobalExceptionFilter());

    // Global interceptors
    app.useGlobalInterceptors(new LoggingInterceptor());
    app.useGlobalInterceptors(new TimeoutInterceptor());

    // CORS
    const configService = app.get(ConfigService);
    app.enableCors({
        origin: configService.get<string>('ALLOWED_ORIGINS', '').split(','),
        credentials: true,
    });

    const port = configService.get<number>('PORT', 3000);
    await app.listen(port);

    logger.log(`Banking API running on port ${port}`);
}

bootstrap();
```

## Module Definition

```typescript
// modules/payments/payments.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { PaymentsController } from './payments.controller';
import { PaymentsService } from './payments.service';
import { PaymentsRepository } from './payments.repository';
import { Payment } from './entities/payment.entity';

@Module({
    imports: [
        TypeOrmModule.forFeature([Payment]),
    ],
    controllers: [PaymentsController],
    providers: [PaymentsService, PaymentsRepository],
    exports: [PaymentsService],  // Available to other modules
})
export class PaymentsModule {}
```

## Controller

```typescript
// modules/payments/payments.controller.ts
import {
    Controller,
    Get,
    Post,
    Body,
    Param,
    Query,
    ParseIntPipe,
    DefaultValuePipe,
    HttpStatus,
} from '@nestjs/common';
import { PaymentsService } from './payments.service';
import { CreatePaymentDto } from './dto/create-payment.dto';
import { PaymentResponseDto } from './dto/payment-response.dto';
import { PaginationDto } from '../../common/dtos/pagination.dto';

@Controller('payments')
export class PaymentsController {
    constructor(private readonly paymentsService: PaymentsService) {}

    @Post()
    async create(
        @Body() createPaymentDto: CreatePaymentDto,
    ): Promise<PaymentResponseDto> {
        const payment = await this.paymentsService.create(createPaymentDto);
        return PaymentResponseDto.fromEntity(payment);
    }

    @Get(':id')
    async findOne(@Param('id') id: string): Promise<PaymentResponseDto> {
        const payment = await this.paymentsService.findOne(id);
        return PaymentResponseDto.fromEntity(payment);
    }

    @Get(':accountId/transactions')
    async findAll(
        @Param('accountId') accountId: string,
        @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
        @Query('limit', new DefaultValuePipe(50), ParseIntPipe) limit: number,
    ) {
        const { items, total } = await this.paymentsService.findAll(
            accountId,
            { page, limit },
        );

        return {
            items: items.map(PaymentResponseDto.fromEntity),
            pagination: {
                page,
                limit,
                total,
                totalPages: Math.ceil(total / limit),
            },
        };
    }
}
```

## Service

```typescript
// modules/payments/payments.service.ts
import {
    Injectable,
    NotFoundException,
    UnprocessableEntityException,
    Logger,
} from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Payment } from './entities/payment.entity';
import { CreatePaymentDto } from './dto/create-payment.dto';

@Injectable()
export class PaymentsService {
    private readonly logger = new Logger(PaymentsService.name);

    constructor(
        @InjectRepository(Payment)
        private readonly paymentRepo: Repository<Payment>,
    ) {}

    async create(dto: CreatePaymentDto): Promise<Payment> {
        this.logger.log(`Creating payment: ${dto.fromAccount} -> ${dto.toAccount}`);

        // Validate accounts exist
        await this.validateAccounts(dto.fromAccount, dto.toAccount);

        // Check sufficient funds
        const balance = await this.getAccountBalance(dto.fromAccount);
        if (balance < dto.amount) {
            throw new UnprocessableEntityException({
                code: 'INSUFFICIENT_FUNDS',
                message: `Available balance: ${balance}, Requested: ${dto.amount}`,
            });
        }

        // Create payment
        const payment = this.paymentRepo.create({
            fromAccount: dto.fromAccount,
            toAccount: dto.toAccount,
            amount: dto.amount,
            currency: dto.currency,
            reference: dto.reference,
            status: 'PENDING',
        });

        return this.paymentRepo.save(payment);
    }

    async findOne(id: string): Promise<Payment> {
        const payment = await this.paymentRepo.findOne({ where: { id } });
        if (!payment) {
            throw new NotFoundException(`Payment ${id} not found`);
        }
        return payment;
    }

    async findAll(accountId: string, pagination: { page: number; limit: number }) {
        const [items, total] = await this.paymentRepo.findAndCount({
            where: [
                { fromAccount: accountId },
                { toAccount: accountId },
            ],
            order: { createdAt: 'DESC' },
            skip: (pagination.page - 1) * pagination.limit,
            take: pagination.limit,
        });

        return { items, total };
    }

    private async validateAccounts(from: string, to: string): Promise<void> {
        if (from === to) {
            throw new UnprocessableEntityException({
                code: 'SELF_PAYMENT',
                message: 'Cannot pay to same account',
            });
        }
        // Check with account service...
    }

    private async getAccountBalance(accountId: string): Promise<number> {
        // Call account service...
        return 0;
    }
}
```

## DTOs with Class-Validator

```typescript
// dto/create-payment.dto.ts
import {
    IsString,
    IsNotEmpty,
    IsNumber,
    Min,
    Max,
    IsOptional,
    Matches,
    Length,
} from 'class-validator';
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';

export class CreatePaymentDto {
    @ApiProperty({ example: 'ACC-001' })
    @IsString()
    @IsNotEmpty()
    @Length(10, 34)
    fromAccount: string;

    @ApiProperty({ example: 'ACC-002' })
    @IsString()
    @IsNotEmpty()
    @Length(10, 34)
    toAccount: string;

    @ApiProperty({ example: 100.00 })
    @IsNumber()
    @Min(0.01)
    @Max(1000000)
    amount: number;

    @ApiProperty({ example: 'USD' })
    @IsString()
    @IsNotEmpty()
    @Matches(/^[A-Z]{3}$/, { message: 'Invalid currency code' })
    currency: string;

    @ApiPropertyOptional({ example: 'Invoice payment' })
    @IsOptional()
    @IsString()
    @Length(0, 140)
    reference?: string;
}
```

## Exception Filter

```typescript
// common/filters/global-exception.filter.ts
import {
    ExceptionFilter,
    Catch,
    ArgumentsHost,
    HttpException,
    HttpStatus,
    Logger,
} from '@nestjs/common';
import { Response, Request } from 'express';

@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
    private readonly logger = new Logger(GlobalExceptionFilter.name);

    catch(exception: unknown, host: ArgumentsHost) {
        const ctx = host.switchToHttp();
        const response = ctx.getResponse<Response>();
        const request = ctx.getRequest<Request>();

        let status = HttpStatus.INTERNAL_SERVER_ERROR;
        let code = 'INTERNAL_ERROR';
        let message = 'An unexpected error occurred';

        if (exception instanceof HttpException) {
            status = exception.getStatus();
            const responseBody = exception.getResponse();

            if (typeof responseBody === 'object' && 'code' in responseBody) {
                code = (responseBody as any).code;
                message = (responseBody as any).message || exception.message;
            } else {
                message = exception.message;
            }
        } else if (exception instanceof Error) {
            this.logger.error(`Unexpected error: ${exception.message}`, exception.stack);
        }

        response.status(status).json({
            error: {
                code,
                message,
                requestId: request.headers['x-correlation-id'],
                timestamp: new Date().toISOString(),
            },
        });
    }
}
```

## Docker Deployment

```dockerfile
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM node:20-alpine AS production

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

COPY --from=builder /app/dist ./dist

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3000

CMD ["node", "dist/main.js"]
```

## Common Mistakes

1. **Putting business logic in controllers**: Controllers should only handle HTTP concerns
2. **No global validation pipe**: Manual validation instead of class-validator
3. **Not using DTOs**: Raw request bodies passed to services
4. **Missing exception filter**: Default NestJS error format
5. **Circular module dependencies**: Module A imports B imports A
6. **No health checks**: Kubernetes cannot determine pod readiness

## Interview Questions

1. How do you structure a NestJS application with 10+ modules?
2. Design a validation pipeline for a complex payment request.
3. How do you handle circular dependencies between modules?
4. What is the difference between a Guard, Interceptor, and Pipe?
5. How do you implement request-scoped services in NestJS?

## Cross-References

- `./type-safety.md` - TypeScript type patterns
- `./testing.md` - NestJS testing patterns
- `./performance.md` - NestJS performance tuning
- `../api-design/README.md` - REST API design
- `../microservices/README.md` - NestJS in microservices
