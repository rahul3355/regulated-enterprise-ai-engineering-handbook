# Distributed Systems for Banking GenAI Platforms

## Why Distributed Systems

A global bank processes millions of transactions across multiple regions. No single machine can handle the load, provide geographic redundancy, or meet regulatory availability requirements. Distributed systems enable horizontal scaling, fault tolerance, and multi-region deployment but introduce complexity in consistency, ordering, and failure handling.

## CAP Theorem

### The Trade-Off

```
CAP Theorem: In a distributed system, you can only guarantee 2 of 3:

- Consistency: Every read receives the most recent write or an error
- Availability: Every request receives a response (without guarantee it's the latest)
- Partition Tolerance: System continues despite network failures between nodes

Banking choice: CP (Consistency + Partition Tolerance)
- Account balances must be consistent
- Payments cannot be processed during network partitions without risk
- Better to be unavailable than inconsistent
```

### Practical Application

```python
# CP system: prefer error over stale data
class AccountService:
    def get_balance(self, account_id: str) -> Decimal:
        # Try primary datacenter
        try:
            return self.primary_db.get_balance(account_id)
        except ConnectionError:
            # Partition detected - return error rather than stale data
            raise ServiceUnavailableError(
                'Unable to reach primary database. '
                'Balance information may be stale.'
            )
            # Alternative: return cached balance with staleness warning

# AP system: prefer availability over consistency (for read-only data)
class BranchLocationService:
    def get_branches(self, city: str) -> list[dict]:
        # Try primary, fall back to cache
        try:
            return self.primary_db.get_branches(city)
        except ConnectionError:
            # Branch locations don't change often - return cached data
            return self.cache.get_branches(city)
```

## Consistency Models

### Strong Consistency

```python
# Linearizable reads - always see latest write
# Requires quorum reads in distributed database

# MongoDB strong consistency
class StrongConsistentAccount:
    def get_balance(self, account_id: str) -> Decimal:
        # Read from primary with majority read concern
        result = db.accounts.find_one(
            {'_id': account_id},
            read_concern=ReadConcern.MAJORITY,
        )
        return Decimal(result['balance'])

    def update_balance(self, account_id: str, amount: Decimal):
        # Write with majority acknowledgment
        db.accounts.update_one(
            {'_id': account_id},
            {'$inc': {'balance': amount}},
            write_concern=WriteConcern(w='majority'),
        )
```

### Eventual Consistency

```python
# Read replicas - may lag behind primary
class EventuallyConsistentAccount:
    def get_transactions(self, account_id: str) -> list[dict]:
        # Read from replica (may be slightly stale)
        return db.accounts.aggregate([
            {'$match': {'account_id': account_id}},
            {'$sort': {'created_at': -1}},
            {'$limit': 50},
        ], read_preference=ReadPreference.SECONDARY)

    def get_recent_transaction(self, account_id: str) -> dict | None:
        """Handle stale reads gracefully."""
        transactions = self.get_transactions(account_id)
        if not transactions:
            # May be stale - check if this is expected
            last_update = self.cache.get(f'account:{account_id}:last_update')
            if last_update:
                staleness = (datetime.utcnow() - last_update).total_seconds()
                if staleness > 300:  # 5 minutes
                    logger.warning(
                        'Transaction data may be stale',
                        extra={'account_id': account_id, 'staleness_seconds': staleness},
                    )
        return transactions[0] if transactions else None
```

### Causal Consistency

```python
# Maintain causality: if A happens before B, all readers see A before B
class CausalConsistentService:
    def __init__(self):
        self.causal_context = {}

    def write_with_context(self, account_id: str, data: dict):
        """Write with causal context."""
        # Include context from previous read
        session = client.start_session(causal_consistency=True)
        session.advance_operation_time(self.causal_context.get(account_id))

        db.accounts.update_one(
            {'_id': account_id},
            {'$set': data},
            session=session,
        )

        # Save context for subsequent reads
        self.causal_context[account_id] = session.operation_time

    def read_with_context(self, account_id: str) -> dict:
        """Read with causal context."""
        session = client.start_session(causal_consistency=True)
        session.advance_operation_time(self.causal_context.get(account_id))

        result = db.accounts.find_one(
            {'_id': account_id},
            session=session,
        )

        self.causal_context[account_id] = session.operation_time
        return result
```

## Consensus Algorithms

### Raft Consensus

```
# Raft ensures all nodes agree on same state through leader election

# Leader handles all writes
# Followers replicate leader's log
# If leader fails, followers elect new leader

# Node states:
# Leader   - handles client requests, replicates log
# Follower - receives log entries from leader, responds to heartbeats
# Candidate - campaigning for leadership

# Election process:
# 1. Follower timeout without heartbeat -> become Candidate
# 2. Candidate requests votes from peers
# 3. Candidate with majority votes becomes Leader
# 4. New leader sends heartbeats to establish authority
```

### Practical Usage: etcd/Consul

```python
# Distributed lock using etcd (Raft-backed)
import etcd3

etcd_client = etcd3.client(host='etcd-cluster', port=2379)

class DistributedLock:
    def __init__(self, key: str, ttl: int = 10):
        self.key = key
        self.lease = etcd_client.lease(ttl)
        self.lock = etcd_client.lock(self.key, ttl=self.lease)

    def acquire(self) -> bool:
        return self.lock.acquire()

    def release(self):
        self.lock.release()

    def __enter__(self):
        self.acquire()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.release()

# Usage: distributed leader election
def leader_election(service_name: str):
    lock = DistributedLock(f'leader:{service_name}', ttl=30)

    if lock.acquire():
        # This instance is the leader
        logger.info(f'Elected as leader for {service_name}')

        # Renew lease periodically
        while True:
            lock.refresh()
            time.sleep(10)
    else:
        logger.info(f'Not elected as leader for {service_name}')
```

## Distributed Transactions

### Two-Phase Commit (2PC)

```python
# 2PC Coordinator
class TwoPhaseCommitCoordinator:
    def execute(self, participants: list[Participant], operation: str) -> bool:
        """
        Phase 1: Prepare
        - Ask all participants if they can commit
        - If any says no, abort

        Phase 2: Commit/Abort
        - If all said yes, tell all to commit
        - If any said no, tell all to abort
        """
        try:
            # Phase 1: Prepare
            votes = []
            for participant in participants:
                vote = participant.prepare(operation)
                votes.append(vote)

            if all(votes):
                # Phase 2: Commit
                for participant in participants:
                    participant.commit(operation)
                return True
            else:
                # Phase 2: Abort
                for participant in participants:
                    participant.abort(operation)
                return False

        except Exception:
            # Abort all participants
            for participant in participants:
                try:
                    participant.abort(operation)
                except Exception:
                    logger.error(f'Failed to abort participant: {participant}')
            raise
```

### Saga Pattern (Preferred over 2PC)

```python
# Choreography-based Saga (event-driven)
# Each service publishes events after local transaction
# Other services react to events

# Step 1: Payment Service creates payment (PENDING)
# Event: payment.created

# Step 2: Account Service debits account
# Event: account.debited

# Step 3: Account Service credits account
# Event: account.credited

# Step 4: Payment Service confirms payment (COMPLETED)
# Event: payment.completed

# Compensation (on failure):
# If Step 2 fails: cancel payment
# If Step 3 fails: credit back source, cancel payment

class TransferSaga:
    def __init__(self):
        self.steps = []
        self.compensations = []
        self.completed_steps = []

    def add_step(self, step_fn, compensation_fn):
        self.steps.append(step_fn)
        self.compensations.append(compensation_fn)

    def execute(self, context: dict):
        for i, step in enumerate(self.steps):
            try:
                step(context)
                self.completed_steps.append(i)
            except Exception:
                # Compensate completed steps in reverse
                for j in reversed(self.completed_steps):
                    self.compensations[j](context)
                raise
```

## Distributed Caching

### Cache Coherence

```python
# Invalidate cache across all nodes when data changes
class DistributedCacheInvalidation:
    def __init__(self, pubsub_client: redis.Redis):
        self.cache = redis.Redis(db=0)
        self.pubsub = redis.Redis(db=1)
        self.pubsub.subscribe('cache_invalidation')

    def invalidate(self, key: str):
        """Invalidate key locally and broadcast to other nodes."""
        self.cache.delete(key)
        self.pubsub.publish('cache_invalidation', json.dumps({'key': key}))

    def listen(self):
        """Listen for invalidation messages from other nodes."""
        for message in self.pubsub.listen():
            if message['type'] == 'message':
                data = json.loads(message['data'])
                self.cache.delete(data['key'])
```

## Clock Synchronization

### Logical Clocks

```python
# Lamport Timestamps for event ordering
class LamportClock:
    def __init__(self):
        self.counter = 0

    def increment(self):
        self.counter += 1
        return self.counter

    def update(self, received_timestamp: int) -> int:
        self.counter = max(self.counter, received_timestamp) + 1
        return self.counter

# Usage: order events across services
def process_payment(event: dict, local_clock: LamportClock):
    remote_ts = event.get('timestamp', 0)
    local_ts = local_clock.update(remote_ts)

    event['lamport_timestamp'] = local_ts
    event['node_id'] = socket.gethostname()

    # Events ordered by (lamport_timestamp, node_id)
    # This gives total ordering across distributed system
```

### Physical Clock Synchronization

```
# NTP (Network Time Protocol) for clock sync
# Required for:
# - Transaction timestamps
# - Audit trail ordering
# - Regulatory compliance timestamps

# Banking requirement:
# - Clocks synchronized within 100ms of UTC
# - Monitored continuously
# - Alerted when drift exceeds threshold

# NTP configuration:
# server ntp1.bank.internal iburst
# server ntp2.bank.internal iburst
# server ntp3.bank.internal iburst
```

## Partition Handling

### Network Partition Detection

```python
class PartitionDetector:
    def __init__(self):
        self.peers = ['node-1', 'node-2', 'node-3']
        self.heartbeat_interval = 5  # seconds
        self.partition_threshold = 15  # 3 missed heartbeats

    def check_connectivity(self) -> dict:
        """Check connectivity to all peers."""
        results = {}
        for peer in self.peers:
            try:
                response = requests.get(f'http://{peer}/health', timeout=3)
                results[peer] = response.status_code == 200
            except Exception:
                results[peer] = False
        return results

    def is_partitioned(self) -> bool:
        """Detect if we're in a network partition."""
        results = self.check_connectivity()
        unreachable = sum(1 for connected in results.values() if not connected)

        # If majority of peers unreachable, likely partitioned
        return unreachable > len(self.peers) // 2
```

### Split-Brain Prevention

```python
# Quorum-based decision making
# Requires majority of nodes to agree before proceeding

class QuorumDecision:
    def __init__(self, total_nodes: int):
        self.total_nodes = total_nodes
        self.quorum_size = (total_nodes // 2) + 1

    def can_proceed(self, votes: dict) -> bool:
        yes_votes = sum(1 for vote in votes.values() if vote)
        return yes_votes >= self.quorum_size

# Usage: only process payments if quorum agrees
quorum = QuorumDecision(total_nodes=5)
votes = health_checker.check_all_nodes()

if quorum.can_proceed(votes):
    process_payments()
else:
    halt_payments()  # Prevent split-brain
    alert_oncall('Network partition detected - halting payments')
```

## GenAI in Distributed Systems

### Distributed Model Serving

```python
# Load balance AI inference across multiple model servers
class DistributedModelServing:
    def __init__(self):
        self.model_servers = [
            'model-server-1:8080',
            'model-server-2:8080',
            'model-server-3:8080',
        ]
        self.request_count = 0

    def round_robin_server(self) -> str:
        server = self.model_servers[self.request_count % len(self.model_servers)]
        self.request_count += 1
        return server

    async def inference(self, prompt: str) -> str:
        server = self.round_robin_server()
        response = await aiohttp.post(
            f'http://{server}/v1/chat/completions',
            json={'prompt': prompt},
            timeout=60,
        )
        return await response.json()
```

### Distributed Embedding Index

```python
# Shard embeddings across multiple nodes for scalability
class DistributedEmbeddingIndex:
    def __init__(self, num_shards: int = 8):
        self.num_shards = num_shards
        self.shards = [EmbeddingIndex() for _ in range(num_shards)]

    def _shard_id(self, text: str) -> int:
        return hash(text) % self.num_shards

    def add_embedding(self, text: str, embedding: list[float]):
        shard = self.shards[self._shard_id(text)]
        shard.add(text, embedding)

    def search(self, query_embedding: list[float], top_k: int = 10) -> list[dict]:
        # Search all shards in parallel
        results = []
        for shard in self.shards:
            shard_results = shard.search(query_embedding, top_k)
            results.extend(shard_results)

        # Merge and sort results
        results.sort(key=lambda x: x['similarity'], reverse=True)
        return results[:top_k]
```

## Common Mistakes

1. **Assuming network is reliable**: No timeout, no retry, no fallback
2. **Ignoring clock skew**: Ordering events without logical clocks
3. **Using 2PC in production**: Locks resources, doesn't scale
4. **No partition handling**: System behavior undefined during network split
5. **Trusting single node**: Leader election not implemented
6. **Ignoring partial failures**: Some nodes succeed, others fail
7. **Not monitoring replication lag**: Reads from stale replicas

## Interview Questions

1. How do you choose between CP and AP for different banking services?
2. Design a system that processes payments correctly even during network partitions.
3. Why is Saga preferred over 2PC for microservices transactions?
4. How do you detect and handle a split-brain scenario?
5. Explain how you'd implement causal consistency for account statements.

## Cross-References

- [[microservices/README.md]] - Distributed data management across services
- [[resilience-patterns/README.md]] - Handling partial failures
- [[queues-and-events/README.md]] - Event sourcing for consistency
- [[caching/README.md]] - Distributed cache coherence
- [[concurrency/README.md]] - Concurrency models for distributed systems
