# Async Processing for Banking GenAI Platforms

## Why Async Processing

Banking systems process millions of transactions that cannot complete synchronously. GenAI inference on large documents, batch payment processing, end-of-day reconciliation, and regulatory reporting all require background execution. Synchronous processing would block user requests, timeout on long operations, and create poor user experiences.

## Core Async Patterns

### Background Job Processing

```python
# Payment batch processing - async job
from celery import Celery

celery_app = Celery('banking_jobs')
celery_app.conf.update(
    broker_url='redis://redis-broker:6379/0',
    result_backend='redis://redis-broker:6379/1',
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='UTC',
    enable_utc=True,
    task_acks_late=True,
    worker_prefetch_multiplier=1,
    task_reject_on_worker_lost=True,
)

@celery_app.task(
    bind=True,
    max_retries=3,
    default_retry_delay=60,
    acks_late=True,
)
def process_payment_batch(self, batch_id: str):
    """Process a batch of 1000+ payments asynchronously."""
    try:
        batch = PaymentBatch.objects.get(id=batch_id)
        batch.status = 'processing'
        batch.save()

        for payment in batch.payments:
            result = execute_payment(payment)
            PaymentResult.objects.create(
                payment=payment,
                status=result.status,
                reference=result.reference,
            )

        batch.status = 'completed'
        batch.completed_at = timezone.now()
        batch.save()

        # Notify clients via webhook
        notify_batch_completion(batch)

    except Exception as exc:
        # Retry with exponential backoff
        raise self.retry(exc=exc, countdown=2 ** self.request.retries * 60)
```

### Job Status Tracking

```python
# Job status API
class JobStatus:
    PENDING = 'pending'
    QUEUED = 'queued'
    RUNNING = 'running'
    SUCCEEDED = 'succeeded'
    FAILED = 'failed'
    CANCELLED = 'cancelled'

@dataclass
class JobRecord:
    job_id: str
    type: str
    status: str
    created_at: datetime
    started_at: datetime | None
    completed_at: datetime | None
    result: dict | None
    error: str | None
    progress: float  # 0.0 to 1.0

# Job status endpoint
GET /v1/jobs/JOB-001
{
  "job_id": "JOB-001",
  "type": "payment_batch",
  "status": "running",
  "progress": 0.45,
  "created_at": "2026-04-10T08:00:00Z",
  "started_at": "2026-04-10T08:00:05Z",
  "estimated_completion": "2026-04-10T08:05:00Z"
}
```

## Worker Pool Architecture

### Worker Configuration

```python
# Worker types for different job categories
# CPU-intensive workers for AI inference
celery_app.conf.task_routes = {
    'ai.document_analysis': {'queue': 'ai-inference'},
    'ai.embedding_generation': {'queue': 'ai-inference'},
    'ai.model_fine_tuning': {'queue': 'ai-training'},

    # I/O-intensive workers for banking operations
    'payment.process_batch': {'queue': 'payments'},
    'notification.send_emails': {'queue': 'notifications'},
    'report.generate_daily': {'queue': 'reports'},

    # Default queue
    '*': {'queue': 'default'},
}

# Worker scaling based on queue depth
# ai-inference queue: 2-20 workers (GPU-constrained)
# payments queue: 5-50 workers (regulated throughput)
# notifications queue: 1-10 workers (email rate limits)
```

### Priority Queue Management

```python
# Priority levels for banking jobs
PRIORITY_CRITICAL = 0   # Fraud detection, real-time alerts
PRIORITY_HIGH = 1       # Payment processing, KYC verification
PRIORITY_NORMAL = 5     # Report generation, data sync
PRIORITY_LOW = 9        # Analytics, cleanup tasks

@celery_app.task(priority=PRIORITY_HIGH)
def process_urgent_payment(payment_id):
    """Urgent payment - processed before normal jobs."""
    pass
```

### Concurrency Control

```python
# Prevent duplicate job execution
from celery.utils.lock import acquire_lock

@celery_app.task(bind=True, max_retries=3)
def generate_daily_report(self, report_date: str):
    """Ensure only one instance runs per date."""
    lock_key = f'daily_report:{report_date}'

    with acquire_lock(lock_key, expire=3600) as acquired:
        if not acquired:
            # Another worker already processing
            return {'status': 'skipped', 'reason': 'lock_held'}

        report = ReportGenerator.generate(report_date)
        return {'status': 'completed', 'report_id': report.id}
```

## GenAI Async Patterns

### Streaming LLM Responses

```python
# Async streaming for GenAI completions
async def stream_ai_response(prompt: str, stream: asyncio.Queue):
    """Stream tokens as they're generated."""
    async for chunk in ai_client.generate_stream(prompt):
        await stream.put({
            'type': 'chunk',
            'content': chunk.text,
            'index': chunk.index,
        })
    await stream.put({'type': 'done'})

# Consumer reads and forwards via SSE
async def sse_handler(request: Request):
    queue = asyncio.Queue()
    task = asyncio.create_task(stream_ai_response(prompt, queue))

    async def event_generator():
        while True:
            event = await queue.get()
            if event['type'] == 'done':
                yield ServerSentEvent(data='{"done": true}', event='done')
                break
            yield ServerSentEvent(
                data=json.dumps(event),
                event='chunk',
            )

    return EventSourceResponse(event_generator())
```

### Batch Document Processing

```python
# Async batch processing for KYC documents
class DocumentProcessor:
    def __init__(self, max_concurrent: int = 5):
        self.semaphore = asyncio.Semaphore(max_concurrent)
        self.results = []

    async def process_documents(self, documents: list[Document]):
        """Process multiple documents with concurrency limit."""
        tasks = [
            self._process_single(doc)
            for doc in documents
        ]
        return await asyncio.gather(*tasks, return_exceptions=True)

    async def _process_single(self, doc: Document):
        async with self.semaphore:
            result = await self.ai_client.analyze(doc.content)
            await self.store_result(doc.id, result)
            return {'document_id': doc.id, 'status': 'success'}
```

## Scheduled Jobs

### Cron-Like Scheduling

```python
# Banking scheduled jobs
from celery.schedules import crontab

celery_app.conf.beat_schedule = {
    'end-of-day-reconciliation': {
        'task': 'reports.generate_eod_reconciliation',
        'schedule': crontab(hour=23, minute=30),
        'options': {'expires': 3600},
    },
    'daily-risk-assessment': {
        'task': 'risk.run_daily_assessment',
        'schedule': crontab(hour=6, minute=0),
    },
    'hourly-metrics-export': {
        'task': 'metrics.export_hourly',
        'schedule': crontab(minute=0),
    },
    'weekly-compliance-report': {
        'task': 'compliance.generate_weekly_report',
        'schedule': crontab(hour=8, minute=0, day_of_week=1),
    },
}
```

## Error Handling and Retry Strategies

### Retry with Exponential Backoff

```python
@celery_app.task(bind=True, max_retries=5)
def call_external_payment_api(self, payment_data):
    try:
        response = requests.post(
            'https://api.external-bank.com/payments',
            json=payment_data,
            timeout=30,
        )
        response.raise_for_status()
        return response.json()

    except requests.exceptions.Timeout:
        # Retry with increasing delay: 60s, 120s, 240s, 480s, 960s
        retry_delay = 60 * (2 ** self.request.retries)
        raise self.retry(countdown=retry_delay)

    except requests.exceptions.HTTPError as e:
        if e.response.status_code == 400:
            # Client error - don't retry
            raise
        elif e.response.status_code == 503:
            # Service unavailable - retry
            raise self.retry(countdown=120)
        else:
            raise
```

### Dead Letter Queue

```python
# Jobs that exhaust retries go to dead letter queue
@celery_app.task(bind=True, max_retries=3)
def process_transaction(self, transaction_id):
    try:
        # Process transaction
        pass
    except Exception as exc:
        if self.request.retries >= self.max_retries:
            # Move to dead letter queue for manual review
            DeadLetterQueue.objects.create(
                task_name=self.name,
                task_id=self.request.id,
                args=self.request.args,
                error=str(exc),
                retry_count=self.request.retries,
            )
            alert_oncall_team(f'Task {self.name} failed permanently')
        raise self.retry(exc=exc)
```

## Monitoring and Observability

### Job Metrics

```python
# Track job metrics for monitoring
class JobMetrics:
    def record_start(self, task_name: str, task_id: str):
        metrics.increment('jobs.started', tags=[f'task:{task_name}'])
        metrics.gauge('jobs.running', 1, tags=[f'task:{task_name}'])

    def record_success(self, task_name: str, duration: float):
        metrics.decrement('jobs.running', tags=[f'task:{task_name}'])
        metrics.histogram('jobs.duration', duration, tags=[f'task:{task_name}'])
        metrics.increment('jobs.succeeded', tags=[f'task:{task_name}'])

    def record_failure(self, task_name: str, error_type: str):
        metrics.increment('jobs.failed', tags=[
            f'task:{task_name}',
            f'error:{error_type}',
        ])
```

### Worker Health Checks

```python
# Worker self-health reporting
@app.task
def worker_health():
    """Check worker health and report issues."""
    return {
        'hostname': socket.gethostname(),
        'pid': os.getpid(),
        'memory_mb': psutil.Process().memory_info().rss / 1024 / 1024,
        'cpu_percent': psutil.cpu_percent(),
        'queue_depth': get_queue_depth(),
        'active_tasks': get_active_task_count(),
    }
```

## Common Mistakes

1. **No idempotency in workers**: Workers process same message twice on retry
2. **Infinite retries**: Missing `max_retries` causes endless retry loops
3. **Blocking I/O in async code**: Using synchronous HTTP calls inside async functions
4. **No job timeout**: Workers hang indefinitely on unresponsive services
5. **Missing dead letter handling**: Failed jobs disappear without trace
6. **Single worker bottleneck**: All jobs routed to same queue, blocking critical work
7. **No progress tracking**: Clients cannot poll job status

## Interview Questions

1. How do you ensure exactly-once processing for a payment job that may be retried?
2. Design a worker pool that prioritizes fraud detection over batch reporting.
3. How do you handle a scenario where 10,000 jobs are queued but only 50 workers are available?
4. What happens when a worker crashes mid-processing? How do you recover?
5. How would you implement graceful shutdown that finishes current jobs?

## Cross-References

- [[queues-and-events/README.md]] - Kafka and event-driven async patterns
- `../python/celery.md` - Celery-specific implementation details
- `../python/async-python.md` - Asyncio patterns and event loops
- `../resilience-patterns/README.md` - Retry and circuit breaker patterns
- `../backend-testing/README.md` - Testing async job processing
