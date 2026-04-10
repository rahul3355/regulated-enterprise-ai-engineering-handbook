# Node.js Performance Engineering

## Event Loop Monitoring

### Detecting Event Loop Blockage

```typescript
import { monitorEventLoopDelay } from 'perf_hooks';

const monitor = monitorEventLoopDelay({ resolution: 10 });
monitor.enable();

// Check every second
setInterval(() => {
    const percentile = monitor.percentile(99);
    if (percentile > 100) {  // > 100ms at P99
        console.warn(`Event loop delay: ${percentile}ms at P99`);
    }
    monitor.reset();
}, 1000);

// Causes of event loop blocking:
// - JSON.parse/stringify on large objects
// - Synchronous file I/O
// - Regex with catastrophic backtracking
// - Large array sorting
// - CPU-intensive computation
```

### Health Check Endpoint

```typescript
import { eventLoopUtilization } from 'perf_hooks';

@Get('health/ready')
async readinessCheck() {
    const elu1 = eventLoopUtilization();
    await new Promise(resolve => setTimeout(resolve, 100));
    const elu2 = eventLoopUtilization(elu1);

    const elu = elu2.utilization;

    // Event loop is overloaded
    if (elu > 0.9) {
        return {
            status: 'degraded',
            eventLoopUtilization: elu,
        };
    }

    return {
        status: 'ready',
        eventLoopUtilization: elu,
    };
}
```

## Profiling

### CPU Profiling

```bash
# Start with profiling enabled
node --prof dist/main.js

# Generate profile after running
node --prof-process isolate-*.log > profile.txt

# Or use clinic.js
npx clinic doctor -- node dist/main.js
npx clinic flame -- node dist/main.js
npx clinic bubbleprof -- node dist/main.js
```

### Memory Profiling

```typescript
// Take heap snapshot
import { writeHeapSnapshot } from 'v8';

@Post('admin/heap-snapshot')
async takeHeapSnapshot() {
    const snapshotPath = `/tmp/heap-${Date.now()}.heapsnapshot`;
    writeHeapSnapshot(snapshotPath);
    return { path: snapshotPath };
}

// Analyze with Chrome DevTools:
// 1. Open Chrome DevTools -> Memory
// 2. Load snapshot file
// 3. Look for objects that shouldn't be retained
```

### Memory Leak Detection

```typescript
// Monitor memory usage
setInterval(() => {
    const usage = process.memoryUsage();
    console.log('Memory usage:', {
        rss: `${(usage.rss / 1024 / 1024).toFixed(2)} MB`,
        heapTotal: `${(usage.heapTotal / 1024 / 1024).toFixed(2)} MB`,
        heapUsed: `${(usage.heapUsed / 1024 / 1024).toFixed(2)} MB`,
        external: `${(usage.external / 1024 / 1024).toFixed(2)} MB`,
    });

    // Alert if heap is growing continuously
    if (usage.heapUsed > 500 * 1024 * 1024) {  // > 500MB
        console.warn('High memory usage detected!');
    }
}, 60000);

// Common Node.js memory leaks:
// 1. Global variables accumulating
// 2. Event listeners not removed
// 3. Closures holding references
// 4. Unbounded caches
// 5. Large objects in request context
```

## Performance Optimization

### Fast JSON Serialization

```typescript
// Standard JSON.stringify is slow for large objects
// Use faster alternatives

// Option 1: fast-json-stringify (pre-compiled schema)
import fastJson from 'fast-json-stringify';

const stringify = fastJson({
    title: 'Payment Response',
    type: 'object',
    properties: {
        id: { type: 'string' },
        amount: { type: 'string' },
        currency: { type: 'string' },
        status: { type: 'string' },
    },
});

const json = stringify(payment);  // 2-5x faster

// Option 2: Use native Response in modern Node.js
return new Response(JSON.stringify(data));
```

### Connection Pooling

```typescript
// HTTP client with connection pooling
import axios from 'axios';
import https from 'https';

const httpClient = axios.create({
    httpsAgent: new https.Agent({
        keepAlive: true,
        maxSockets: 50,
        maxFreeSockets: 10,
        timeout: 30000,
    }),
    timeout: 30000,
});

// Reuse client instance - don't create per request
```

### Caching

```typescript
import NodeCache from 'node-cache';

// In-process cache for frequently accessed data
const cache = new NodeCache({
    stdTTL: 300,       // 5 minutes default TTL
    maxKeys: 10000,    // Limit cache size
    checkperiod: 60,   // Check for expired keys every 60s
});

// Usage
async function getAccountBalance(accountId: string): Promise<number> {
    const cacheKey = `balance:${accountId}`;

    // Check cache
    const cached = cache.get<number>(cacheKey);
    if (cached !== undefined) {
        return cached;
    }

    // Fetch from database
    const balance = await db.getBalance(accountId);

    // Cache for 5 minutes
    cache.set(cacheKey, balance, 300);

    return balance;
}
```

### Stream Processing

```typescript
// Stream large responses instead of loading into memory
import { Readable } from 'stream';

@Get('accounts/:id/statements')
async getStatement(@Param('id') accountId: string, @Res() res: Response) {
    // Create readable stream from database
    const stream = this.statementService.streamStatements(accountId);

    res.setHeader('Content-Type', 'application/json');
    res.setHeader('Transfer-Encoding', 'chunked');

    // Stream directly to response
    stream.pipe(res);
}

// Stream processing for large file uploads
@Post('documents')
@UseInterceptors(FileInterceptor('file'))
async uploadDocument(@UploadedFile() file: Express.Multer.File) {
    // Process file as stream, don't load into memory
    const fileStream = createReadStream(file.path);

    const result = await this.documentService.analyzeStream(fileStream);

    return result;
}
```

## Worker Threads for CPU-Bound Work

```typescript
import { Worker, isMainThread, parentPort, workerData } from 'worker_threads';

if (isMainThread) {
    // Main thread: distribute work to workers
    export async function computeRiskScores(documents: Document[]): Promise<number[]> {
        const workers: Worker[] = [];
        const results: number[] = [];

        const chunkSize = Math.ceil(documents.length / 4);
        const chunks = Array.from({ length: 4 }, (_, i) =>
            documents.slice(i * chunkSize, (i + 1) * chunkSize)
        );

        const promises = chunks.map((chunk, i) => {
            return new Promise<number[]>((resolve, reject) => {
                const worker = new Worker(__filename, { workerData: chunk });

                worker.on('message', resolve);
                worker.on('error', reject);
                worker.on('exit', (code) => {
                    if (code !== 0) reject(new Error(`Worker stopped with exit code ${code}`));
                });
            });
        });

        const allResults = await Promise.all(promises);
        return allResults.flat();
    }
} else {
    // Worker thread: compute on chunk
    const documents = workerData;
    const scores = documents.map(doc => calculateRiskScore(doc));
    parentPort!.postMessage(scores);
}
```

## Load Testing

```typescript
// k6 load test script
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
    stages: [
        { duration: '30s', target: 100 },   // Ramp up to 100 users
        { duration: '1m', target: 100 },    // Stay at 100 users
        { duration: '30s', target: 500 },   // Spike to 500 users
        { duration: '1m', target: 500 },    // Stay at 500 users
        { duration: '30s', target: 0 },     // Ramp down
    ],
    thresholds: {
        http_req_duration: ['p(95)<500'],   // 95% of requests under 500ms
        http_req_failed: ['rate<0.01'],      // Error rate under 1%
    },
};

export default function () {
    const res = http.get('http://localhost:3000/api/v1/accounts/ACC-001');

    check(res, {
        'status is 200': (r) => r.status === 200,
        'response time < 200ms': (r) => r.timings.duration < 200,
    });

    sleep(1);
}
```

## Production Tuning

```bash
# Node.js flags for production
NODE_OPTIONS="--max-old-space-size=4096"  # 4GB heap
node dist/main.js

# Or inline
node --max-old-space-size=4096 dist/main.js

# For low-latency services
node --max-old-space-size=2048 \
     --optimize-for-size \
     dist/main.js

# Monitor with PM2
pm2 start dist/main.js -i max  # Cluster mode (one process per CPU core)
pm2 monit                       # Monitor memory and CPU
```

## Common Mistakes

1. **Blocking the event loop**: CPU-bound work in request handlers
2. **No memory limits**: Node.js crashes with OOM
3. **Not monitoring event loop**: Latency spikes undetected
4. **Loading large files into memory**: Not using streams
5. **Creating HTTP clients per request**: No connection reuse
6. **Unbounded caches**: Memory grows without limit
7. **No load testing**: Surprised by performance under load

## Interview Questions

1. How do you detect and fix event loop blocking in production?
2. Design a system that processes 1GB file uploads without loading into memory.
3. How do you handle CPU-intensive work in a Node.js backend service?
4. What tools do you use to profile memory leaks in Node.js?
5. How do you configure Node.js for a 4GB container deployment?

## Cross-References

- `./express-nestjs.md` - NestJS performance configuration
- `./type-safety.md` - Type-safe performance patterns
- `../performance/README.md` - Overall performance engineering
- `../caching/README.md` - Caching for performance
- `../concurrency/README.md` - Concurrency in Node.js
