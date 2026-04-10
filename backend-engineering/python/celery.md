# Celery Task Queues for Banking

## Why Celery

Celery is the standard Python distributed task queue. It handles background processing for payments, report generation, email notifications, and AI inference. Celery provides retries, monitoring, and horizontal scaling that asyncio alone cannot provide.

## Architecture

```
Producer (FastAPI) -> Broker (Redis/RabbitMQ) -> Worker (Celery) -> Result Backend (Redis)

- Broker: Message queue that holds tasks
- Worker: Process that executes tasks
- Result Backend: Stores task results
- Beat: Scheduler for periodic tasks
```

## Production Configuration

```python
# celery_config.py
from celery import Celery
from celery.schedules import crontab

celery_app = Celery('banking_tasks')

# Broker configuration (Redis)
celery_app.conf.broker_url = 'redis://redis-broker:6379/0'
celery_app.conf.result_backend = 'redis://redis-broker:6379/1'

# Serialization
celery_app.conf.task_serializer = 'json'
celery_app.conf.accept_content = ['json']
celery_app.conf.result_serializer = 'json'

# Timezone
celery_app.conf.timezone = 'UTC'
celery_app.conf.enable_utc = True

# Task acknowledgment
celery_app.conf.task_acks_late = True  # Acknowledge after task completes
celery_app.conf.worker_prefetch_multiplier = 1  # One task at a time

# Task rejection on worker lost
celery_app.conf.task_reject_on_worker_lost = True

# Task time limits
celery_app.conf.task_soft_time_limit = 300   # 5 min soft limit (raises SoftTimeLimitExceeded)
celery_app.conf.task_time_limit = 600        # 10 min hard limit (kills worker)

# Retry settings
celery_app.conf.broker_connection_retry_on_startup = True
celery_app.conf.broker_connection_max_retries = 10

# Task routing
celery_app.conf.task_routes = {
    'tasks.payment.*': {'queue': 'payments'},
    'tasks.reports.*': {'queue': 'reports'},
    'tasks.notifications.*': {'queue': 'notifications'},
    'tasks.ai.*': {'queue': 'ai-inference'},
    'tasks.compliance.*': {'queue': 'compliance'},
}
```

## Task Definition

```python
# tasks/payments.py
from celery import Task
from decimal import Decimal
import logging

logger = logging.getLogger(__name__)


class BasePaymentTask(Task):
    """Base task with payment-specific error handling."""
    max_retries = 3
    default_retry_delay = 60
    acks_late = True

    def on_failure(self, exc, task_id, args, kwargs, einfo):
        """Log payment task failures."""
        logger.error(
            f'Payment task {task_id} failed: {exc}',
            extra={
                'task_id': task_id,
                'args': args,
                'error': str(exc),
            },
        )
        # Alert oncall for payment failures
        alert_oncall_team(f'Payment task failed: {exc}')


@celery_app.task(
    bind=True,
    base=BasePaymentTask,
    name='tasks.payment.process_transfer',
)
def process_transfer(
    self,
    from_account: str,
    to_account: str,
    amount: str,  # Serialize as string for Decimal precision
    currency: str,
    reference: str = None,
) -> dict:
    """Process a bank transfer asynchronously."""
    try:
        amount_decimal = Decimal(amount)

        # Execute transfer
        result = execute_bank_transfer(
            from_account=from_account,
            to_account=to_account,
            amount=amount_decimal,
            currency=currency,
            reference=reference,
        )

        # Send confirmation notification
        send_notification_task.delay(
            account_id=from_account,
            type='transfer_completed',
            details={
                'amount': str(amount_decimal),
                'currency': currency,
                'to': to_account,
            },
        )

        return {
            'status': 'completed',
            'transfer_id': result['id'],
        }

    except InsufficientFundsError as e:
        # Don't retry - permanent error
        logger.warning(f'Insufficient funds: {e}')
        return {'status': 'failed', 'reason': 'insufficient_funds'}

    except (ConnectionError, TimeoutError) as e:
        # Retry on transient errors
        countdown = 60 * (2 ** self.request.retries)  # Exponential backoff
        raise self.retry(exc=e, countdown=countdown)


@celery_app.task(bind=True, base=BasePaymentTask)
def process_payment_batch(self, batch_id: str) -> dict:
    """Process a batch of payments."""
    batch = PaymentBatch.objects.get(id=batch_id)
    batch.status = 'processing'
    batch.save()

    results = []
    for payment in batch.payments:
        try:
            result = process_transfer.delay(
                from_account=payment.from_account,
                to_account=payment.to_account,
                amount=str(payment.amount),
                currency=payment.currency,
                reference=payment.reference,
            )
            results.append({'payment_id': payment.id, 'task_id': result.id})
        except Exception as e:
            results.append({'payment_id': payment.id, 'error': str(e)})

    batch.status = 'completed'
    batch.completed_at = timezone.now()
    batch.results = results
    batch.save()

    return {'batch_id': batch_id, 'total': len(results)}
```

## Retry Strategies

```python
# Exponential backoff with jitter
@celery_app.task(bind=True, max_retries=5)
def call_external_api(self, payment_id: str):
    try:
        response = requests.post(
            'https://api.external.com/payments',
            json={'payment_id': payment_id},
            timeout=30,
        )
        response.raise_for_status()
        return response.json()

    except requests.exceptions.Timeout:
        retry_delay = min(60 * (2 ** self.request.retries), 3600)  # Max 1 hour
        # Add jitter
        import random
        retry_delay = retry_delay * random.uniform(0.5, 1.5)
        raise self.retry(countdown=retry_delay)

    except requests.exceptions.HTTPError as e:
        status_code = e.response.status_code
        if status_code >= 500:
            # Server error - retry
            raise self.retry(countdown=120)
        elif status_code == 429:
            # Rate limited - retry after header value
            retry_after = int(e.response.headers.get('Retry-After', 60))
            raise self.retry(countdown=retry_after)
        else:
            # Client error - don't retry
            raise


# Dead letter queue for failed tasks
@celery_app.task(bind=True, max_retries=3)
def process_critical_task(self, data: dict):
    try:
        result = execute_critical_operation(data)
        return result

    except Exception as exc:
        if self.request.retries >= self.max_retries:
            # Move to dead letter queue
            DeadLetterQueue.objects.create(
                task_name=self.name,
                task_id=self.request.id,
                args=data,
                error=str(exc),
                retry_count=self.request.retries,
                created_at=timezone.now(),
            )
            # Alert oncall team
            alert_oncall_team(
                f'Task {self.name} exhausted retries. '
                f'Moved to dead letter queue.'
            )
        raise self.retry(exc=exc)
```

## Scheduled Tasks

```python
# Periodic tasks with Celery Beat
celery_app.conf.beat_schedule = {
    # Daily reconciliation
    'daily-reconciliation': {
        'task': 'tasks.reconciliation.run_daily',
        'schedule': crontab(hour=23, minute=30),
        'options': {'expires': 3600},
    },

    # Hourly metrics export
    'hourly-metrics': {
        'task': 'tasks.metrics.export_hourly',
        'schedule': crontab(minute=0),
    },

    # Weekly compliance report
    'weekly-compliance-report': {
        'task': 'tasks.compliance.generate_weekly_report',
        'schedule': crontab(hour=8, minute=0, day_of_week=1),
    },

    # Every 5 minutes: check for pending payments
    'process-pending-payments': {
        'task': 'tasks.payment.process_pending',
        'schedule': crontab(minute='*/5'),
    },

    # Monthly statement generation
    'monthly-statements': {
        'task': 'tasks.reports.generate_monthly_statements',
        'schedule': crontab(hour=2, minute=0, day_of_month=1),
    },
}
```

## Monitoring

```python
# Flower monitoring (Celery web dashboard)
# Run: celery -A celery_app flower --port=5555

# Custom monitoring with Prometheus
from celery.signals import (
    task_prerun,
    task_postrun,
    task_failure,
    task_retry,
)
from prometheus_client import Histogram, Counter

TASK_DURATION = Histogram('celery_task_duration_seconds', 'Task duration', ['task_name'])
TASK_SUCCESS = Counter('celery_task_success_total', 'Successful tasks', ['task_name'])
TASK_FAILURE = Counter('celery_task_failure_total', 'Failed tasks', ['task_name'])
TASK_RETRY = Counter('celery_task_retry_total', 'Retried tasks', ['task_name'])


@task_prerun.connect
def task_prerun_handler(task_id, task, *args, **kwargs):
    task._start_time = time.time()


@task_postrun.connect
def task_postrun_handler(task_id, task, *args, retval=None, **kwargs):
    duration = time.time() - getattr(task, '_start_time', time.time())
    TASK_DURATION.labels(task_name=task.name).observe(duration)
    TASK_SUCCESS.labels(task_name=task.name).inc()


@task_failure.connect
def task_failure_handler(task_id, task, exception=None, **kwargs):
    TASK_FAILURE.labels(task_name=task.name).inc()
    logger.error(f'Task {task.name} failed: {exception}')


@task_retry.connect
def task_retry_handler(task_id, task, reason=None, **kwargs):
    TASK_RETRY.labels(task_name=task.name).inc()
    logger.warning(f'Task {task.name} retrying: {reason}')
```

## Worker Configuration

```bash
# Production worker command
celery -A celery_config worker \
    --loglevel=info \
    --concurrency=10 \
    --pool=prefork \
    --queues=payments,reports,notifications \
    --hostname=worker-%h \
    --max-tasks-per-child=1000 \
    --max-memory-per-child=500000  # 500MB

# AI inference worker (separate, GPU-enabled)
celery -A celery_config worker \
    --loglevel=info \
    --concurrency=2 \
    --pool=prefork \
    --queues=ai-inference \
    --hostname=ai-worker-%h \
    --max-tasks-per-child=100
```

## Common Mistakes

1. **Not setting task_acks_late**: Tasks lost if worker crashes during execution
2. **Infinite retries**: Missing max_retries causes endless retry loops
3. **Passing ORM objects**: Pass IDs, not objects (objects become stale)
4. **No task time limits**: Workers hang on unresponsive external services
5. **Single queue for all tasks**: Critical payments blocked by batch reports
6. **Not monitoring task failures**: Silent failures go unnoticed
7. **Missing dead letter handling**: Failed tasks disappear without trace

## Interview Questions

1. How do you ensure a payment task is executed exactly once?
2. Design a retry strategy for an external payment gateway with rate limiting.
3. What happens when a Celery worker crashes mid-task?
4. How do you handle a scenario where 10,000 tasks are queued but only 10 workers are available?
5. How do you migrate tasks from one queue to another during deployment?

## Cross-References

- `./async-python.md` - Async vs Celery: when to use each
- `../async-processing/README.md` - Background job processing patterns
- `../queues-and-events/README.md` - Kafka vs Celery for event processing
- `../backend-testing/README.md` - Testing Celery tasks
