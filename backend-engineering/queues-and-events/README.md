# Queues and Events for Banking GenAI Platforms

## Why Event-Driven Architecture

Banking systems are inherently event-driven: payments occur, balances change, fraud alerts trigger, and regulatory events must be captured. Kafka provides a distributed, fault-tolerant event backbone that decouples producers from consumers, enables replay, and supports real-time processing at scale. For GenAI platforms, events trigger document analysis, model retraining, and inference pipelines.

## Apache Kafka Fundamentals

### Core Concepts

```
Producer -> Topic -> Partition -> Consumer Group -> Consumer

- Topic: Logical channel (e.g., "payments.created")
- Partition: Ordered, immutable sequence of records
- Consumer Group: Set of consumers that share workload
- Offset: Position of consumer within partition
- Broker: Kafka server instance
- Cluster: Group of brokers
```

### Topic Design for Banking

```python
# Topic naming convention: <domain>.<entity>.<event>
TOPICS = {
    # Payment events
    'payments.created': 'Payment initiation events',
    'payments.completed': 'Payment settlement events',
    'payments.failed': 'Payment failure events',
    'payments.reversed': 'Payment reversal events',

    # Account events
    'accounts.balance_updated': 'Balance change events',
    'accounts.created': 'New account events',
    'accounts.closed': 'Account closure events',

    # Fraud events
    'fraud.detected': 'Fraud detection events',
    'fraud.reviewed': 'Fraud review completion',

    # Compliance events
    'compliance.kyc_completed': 'KYC verification events',
    'compliance.aml_flagged': 'AML alert events',
    'compliance.report_generated': 'Regulatory report events',

    # GenAI events
    'ai.document_analyzed': 'Document analysis completion',
    'ai.model_deployed': 'Model deployment events',
    'ai.inference_completed': 'Inference result events',
}
```

### Kafka Producer Configuration

```python
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['kafka-1:9092', 'kafka-2:9092', 'kafka-3:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8'),
    key_serializer=lambda v: v.encode('utf-8'),
    acks='all',  # Wait for all ISR to acknowledge
    retries=3,
    max_in_flight_requests_per_connection=1,  # Ordering guarantee
    enable_idempotence=True,  # Exactly-once producer
    compression_type='lz4',
    batch_size=16384,
    linger_ms=5,
)

# Produce with partition key for ordering
def publish_payment_event(event: dict):
    key = event['account_id']  # Same account -> same partition
    producer.send(
        topic='payments.created',
        key=key,
        value=event,
    )
    producer.flush()
```

### Kafka Consumer Configuration

```python
from kafka import KafkaConsumer

consumer = KafkaConsumer(
    'payments.created',
    'payments.completed',
    bootstrap_servers=['kafka-1:9092'],
    group_id='payment-processor-service',
    auto_offset_reset='earliest',  # Start from beginning
    enable_auto_commit=False,  # Manual commit for reliability
    max_poll_records=100,
    max_poll_interval_ms=300000,  # 5 min processing timeout
    session_timeout_ms=10000,
    heartbeat_interval_ms=3000,
    isolation_level='read_committed',  # Transactional reads
)

for message in consumer:
    event = json.loads(message.value)
    try:
        process_payment_event(event)
        consumer.commit()  # Commit after successful processing
    except Exception:
        # Don't commit - will reprocess on restart
        log_error(event, message.offset)
        raise
```

## Message Ordering Guarantees

### Ordering within Partitions

Kafka guarantees ordering **within a partition**, not across a topic. Use partition keys to ensure related events land in the same partition:

```python
# Correct: All events for same account in same partition
def publish_account_event(account_id: str, event: dict):
    producer.send(
        topic='accounts.balance_updated',
        key=account_id,  # Partition key = account_id
        value=event,
    )
# Result: All balance updates for ACC-001 are ordered

# Incorrect: No partition key means round-robin distribution
def publish_unordered_event(event: dict):
    producer.send(
        topic='accounts.balance_updated',
        value=event,  # No key -> random partition
    )
# Result: Balance updates may arrive out of order
```

### When Ordering Matters

| Scenario | Ordering Required | Partition Key |
|----------|------------------|---------------|
| Payment state transitions | Yes | payment_id |
| Account balance updates | Yes | account_id |
| Fraud detection events | Yes | account_id |
| Log aggregation | No | None |
| Metrics collection | No | None |
| Audit trail events | Yes | entity_id |

## Exactly-Once Semantics

### Producer-Side Exactly-Once

```python
# Idempotent producer configuration
producer = KafkaProducer(
    enable_idempotence=True,
    acks='all',
    max_in_flight_requests_per_connection=1,
)

# Kafka assigns sequence numbers to prevent duplicates
# Even with retries, each message delivered exactly once
```

### End-to-End Exactly-Once

```python
# Kafka Transactions for atomic multi-topic writes
from kafka import KafkaProducer

producer = KafkaProducer(
    transactional_id='payment-processor-1',
    enable_idempotence=True,
    acks='all',
)

producer.init_transactions()

def process_payment_with_events(payment: dict):
    try:
        producer.begin_transaction()

        # Write to database
        db.save_payment(payment)

        # Publish events atomically
        producer.send(
            topic='payments.created',
            key=payment['id'],
            value=payment,
        )
        producer.send(
            topic='audit.payment_created',
            key=payment['id'],
            value={'action': 'payment_created', 'payment_id': payment['id']},
        )

        producer.commit_transaction()

    except Exception:
        producer.abort_transaction()
        raise
```

### Consumer-Side Idempotency

```python
# Idempotent event processing
class PaymentEventProcessor:
    def __init__(self):
        self.processed_ids = set()

    def process(self, event: dict):
        event_id = event['event_id']

        # Check for duplicate
        if event_id in self.processed_ids:
            return  # Already processed

        # Process event
        self.apply_event(event)

        # Mark as processed
        self.processed_ids.add(event_id)

    def apply_event(self, event: dict):
        # Apply to current state
        pass
```

## Event Sourcing

### Event Store Pattern

```python
# Event-sourced account aggregate
class AccountEventSourcing:
    def __init__(self, account_id: str):
        self.account_id = account_id
        self.balance = Decimal('0.00')
        self.status = 'pending'
        self.version = 0
        self.events = []

    def apply_event(self, event: dict):
        event_type = event['type']

        if event_type == 'account_created':
            self.status = 'active'
        elif event_type == 'deposit_made':
            self.balance += event['amount']
        elif event_type == 'withdrawal_made':
            self.balance -= event['amount']
        elif event_type == 'account_closed':
            self.status = 'closed'

        self.version = event['version']
        self.events.append(event)

    def deposit(self, amount: Decimal):
        if self.status != 'active':
            raise ValueError('Account not active')
        if amount <= 0:
            raise ValueError('Amount must be positive')

        event = {
            'type': 'deposit_made',
            'account_id': self.account_id,
            'amount': amount,
            'version': self.version + 1,
            'timestamp': datetime.utcnow().isoformat(),
        }
        self.apply_event(event)
        return event

    def get_balance(self) -> Decimal:
        # Replay all events to compute balance
        return self.balance
```

### Event Replay and State Reconstruction

```python
# Rebuild account state from event stream
def rebuild_account(account_id: str) -> AccountEventSourcing:
    account = AccountEventSourcing(account_id)
    events = event_store.get_events(account_id)

    for event in events:
        account.apply_event(event)

    return account

# Snapshot optimization: store state every N events
def create_snapshot(account: AccountEventSourcing):
    snapshot = {
        'account_id': account.account_id,
        'balance': str(account.balance),
        'status': account.status,
        'version': account.version,
    }
    snapshot_store.save(snapshot)

# Load from snapshot, replay remaining events
def load_account(account_id: str) -> AccountEventSourcing:
    snapshot = snapshot_store.get(account_id)
    account = AccountEventSourcing(account_id)

    if snapshot:
        account.balance = Decimal(snapshot['balance'])
        account.status = snapshot['status']
        account.version = snapshot['version']
        events = event_store.get_events(account_id, from_version=snapshot['version'])
    else:
        events = event_store.get_events(account_id)

    for event in events:
        account.apply_event(event)

    return account
```

## Schema Evolution

### Schema Registry

```python
# Avro schema for payment events
payment_event_schema = {
    "type": "record",
    "name": "PaymentEvent",
    "namespace": "com.bank.payments",
    "fields": [
        {"name": "event_id", "type": "string"},
        {"name": "payment_id", "type": "string"},
        {"name": "account_id", "type": "string"},
        {"name": "amount", "type": {"type": "bytes", "logicalType": "decimal", "precision": 19, "scale": 2}},
        {"name": "currency", "type": "string"},
        {"name": "status", "type": {"type": "enum", "name": "PaymentStatus", "symbols": ["PENDING", "COMPLETED", "FAILED", "REVERSED"]}},
        {"name": "created_at", "type": {"type": "long", "logicalType": "timestamp-millis"}},
        {"name": "reference", "type": ["null", "string"], "default": None},
    ],
}

# Backward compatible evolution:
# - Adding fields with defaults: BACKWARD compatible
# - Removing fields with defaults: FORWARD compatible
# - Renaming fields: NOT compatible (add new field, deprecate old)
```

### Schema Versioning Strategy

```
BACKWARD compatibility (default):
  - New consumers can read old data
  - Add optional fields with defaults
  - Never remove fields (mark deprecated)

FORWARD compatibility:
  - Old consumers can read new data
  - Remove fields (new producers stop writing)

FULL compatibility:
  - Both old and new consumers/producers work
  - Only add optional fields
```

## Consumer Group Management

### Scaling Consumers

```
# 1 partition -> 1 consumer per group
# 6 partitions -> max 6 consumers per group
# More consumers than partitions = idle consumers

# Partition assignment for 3 consumers, 6 partitions:
# Consumer-1: Partitions 0, 1
# Consumer-2: Partitions 2, 3, 4
# Consumer-3: Partitions 5

# Rebalance occurs when:
# - Consumer joins or leaves group
# - Partition count changes
# - Consumer timeout (max.poll.interval.ms exceeded)
```

### Handling Rebalance

```python
from kafka import KafkaConsumer
from kafka.coordinator.assignors.roundrobin import RoundRobinPartitionAssignor

class GracefulRebalanceListener:
    def on_partitions_revoked(self, revoked):
        """Called before rebalance - commit offsets."""
        print(f'Partitions revoked: {revoked}')
        consumer.commit()

    def on_partitions_assigned(self, assigned):
        """Called after rebalance - resume processing."""
        print(f'Partitions assigned: {assigned}')

consumer = KafkaConsumer(
    'payments.created',
    group_id='payment-processor',
    partition_assignment_strategy=[RoundRobinPartitionAssignor],
)

# Subscribe with listener
consumer.subscribe(
    topics=['payments.created'],
    listener=GracefulRebalanceListener(),
)
```

## Banking-Specific Event Patterns

### Audit Trail

```python
# Every state change produces an audit event
def transfer_funds(from_account, to_account, amount):
    events = []

    # Debit event
    events.append({
        'type': 'debit_recorded',
        'account_id': from_account,
        'amount': amount,
        'correlation_id': generate_uuid(),
    })

    # Credit event
    events.append({
        'type': 'credit_recorded',
        'account_id': to_account,
        'amount': amount,
        'correlation_id': events[0]['correlation_id'],
    })

    # Publish both events atomically
    publish_events(events)
```

### Real-Time Fraud Detection

```python
# Stream processing for fraud detection
def fraud_detection_consumer():
    consumer = KafkaConsumer('payments.created')

    for message in consumer:
        payment = json.loads(message.value)

        # Check velocity: transactions in last 1 hour
        recent_count = redis.incr(f'payment_velocity:{payment["account_id"]}')
        redis.expire(f'payment_velocity:{payment["account_id"]}', 3600)

        if recent_count > 50:
            # Publish fraud event
            producer.send('fraud.detected', key=payment['account_id'], value={
                'account_id': payment['account_id'],
                'reason': 'high_velocity',
                'transaction_count': recent_count,
                'payment_id': payment['id'],
            })
```

## GenAI Event Patterns

### Model Inference Event Pipeline

```python
# Event-driven AI inference pipeline
# 1. Document uploaded -> triggers analysis
# 2. Analysis complete -> triggers risk scoring
# 3. Risk score computed -> triggers decision

# Consumer 1: Document analysis
consumer.subscribe(['documents.uploaded'])
for msg in consumer:
    doc = json.loads(msg.value)
    result = ai_analyze(doc)
    producer.send('ai.document_analyzed', key=doc['id'], value=result)

# Consumer 2: Risk scoring
consumer.subscribe(['ai.document_analyzed'])
for msg in consumer:
    analysis = json.loads(msg.value)
    risk = risk_score(analysis)
    producer.send('ai.risk_scored', key=analysis['document_id'], value=risk)

# Consumer 3: Decision engine
consumer.subscribe(['ai.risk_scored'])
for msg in consumer:
    risk = json.loads(msg.value)
    decision = make_decision(risk)
    producer.send('compliance.decision_made', key=risk['document_id'], value=decision)
```

## Common Mistakes

1. **No partition key**: Events arrive out of order for the same entity
2. **Auto-commit enabled**: Messages lost if consumer crashes after commit
3. **Large message sizes**: Kafka not designed for payloads > 1MB
4. **No schema registry**: Consumers break when producers change format
5. **Ignoring consumer lag**: Falling behind creates data staleness
6. **Single broker**: No fault tolerance, single point of failure
7. **Mixing event types**: One topic for unrelated events creates consumer coupling

## Interview Questions

1. How do you guarantee message ordering for account balance updates?
2. Design exactly-once processing for a payment event consumer.
3. What happens when a consumer group rebalances during processing?
4. How do you handle schema evolution when 20 services consume your events?
5. Explain the difference between `acks=1` and `acks=all` for banking events.
6. How would you replay 6 months of payment events to rebuild a view?

## Cross-References

- [[async-processing/README.md]] - Background job processing patterns
- [[microservices/README.md]] - Event-driven microservices communication
- [[distributed-systems/README.md]] - Distributed consistency and event sourcing
- `../python/celery.md` - Task queue vs event streaming
- `../python/async-python.md` - Async event processing patterns
