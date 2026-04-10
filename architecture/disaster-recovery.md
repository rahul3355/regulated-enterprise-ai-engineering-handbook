# Disaster Recovery Architecture

## Overview

Disaster Recovery (DR) ensures that the banking GenAI platform can recover from catastrophic failures -- region outages, data center failures, ransomware, or supply chain compromises. In banking, DR is not optional: regulators require documented DR plans with tested RTO (Recovery Time Objective) and RPO (Recovery Point Objective) targets.

For GenAI systems, DR is uniquely complex because:
- **State is distributed**: Vector databases, caches, message queues, and relational databases all hold state
- **External dependencies**: LLM provider outages are outside the bank's control
- **Model artifacts**: Fine-tuned models and embeddings must be recoverable
- **Compliance continuity**: Audit logs must survive disasters for regulatory reporting

---

## DR Architecture

```mermaid
flowchart TD
    subgraph "Primary Region (us-east-1)"
        A[API Gateway] --> B[RAG Service]
        B --> C[(Vector DB Primary)]
        B --> D[(PostgreSQL Primary)]
        B --> E[(Redis Primary)]
        B --> F[LLM Provider]
    end

    subgraph "DR Region (us-west-2)"
        G[API Gateway Standby] --> H[RAG Service Standby]
        H --> I[(Vector DB Replica)]
        H --> J[(PostgreSQL Replica)]
        H --> K[(Redis Replica)]
        H --> L[LLM Provider]
    end

    C -.->|Async Replication| I
    D -.->|Stream Replication| J
    E -.->|Async Replication| K

    M[Route 53 Health Checks] -->|Failover| G
```

---

## RTO and RPO Targets

| Component | RTO | RPO | Strategy |
|---|---|---|---|
| API Gateway | 5 min | 0 (stateless) | Active-active across regions |
| RAG Query Service | 10 min | 0 (stateless) | Auto-scale in DR region |
| Vector Database | 30 min | 5 min | Async cross-region replication |
| PostgreSQL | 15 min | 0 (sync) | Synchronous streaming replication |
| Redis Cache | 10 min | 5 min (acceptable loss) | Async replication, warm cache |
| Message Broker | 15 min | 0 | Mirrored queues across regions |
| Audit Logs | 1 hour | 0 (no loss acceptable) | Synchronous multi-region writes |
| Document Store (S3) | 15 min | 0 | Cross-region replication (built-in) |

---

## Failover Automation

```python
# dr/failover_manager.py
"""
Automated failover manager for the banking GenAI platform.
Coordinates failover across all components in the correct order.
"""
from dataclasses import dataclass
from enum import Enum
from typing import List, Callable, Awaitable
import asyncio
import logging

logger = logging.getLogger("dr.failover")

class FailoverStatus(Enum):
    HEALTHY = "healthy"
    DEGRADED = "degraded"
    FAILING_OVER = "failing_over"
    FAILOVER_COMPLETE = "failover_complete"
    FAILED = "failed"

@dataclass
class FailoverStep:
    name: str
    action: Callable[..., Awaitable[None]]
    rollback: Callable[..., Awaitable[None]]
    timeout_seconds: float = 60.0
    critical: bool = True  # If True, failure aborts entire failover

class FailoverManager:
    """Manage disaster recovery failover and failback."""

    def __init__(self):
        self.status = FailoverStatus.HEALTHY
        self.failover_history = []

    async def initiate_failover(self, reason: str = "automatic") -> bool:
        """
        Execute failover from primary to DR region.
        Steps are executed in order; if a critical step fails, rollback is attempted.
        """
        self.status = FailoverStatus.FAILING_OVER
        logger.info(f"Initiating failover: {reason}")

        steps = self._get_failover_steps()
        completed_steps = []

        try:
            for step in steps:
                logger.info(f"Executing failover step: {step.name}")
                try:
                    await asyncio.wait_for(step.action(), timeout=step.timeout_seconds)
                    completed_steps.append(step)
                    logger.info(f"Completed step: {step.name}")
                except Exception as e:
                    if step.critical:
                        logger.error(f"Critical step failed: {step.name}: {e}")
                        await self._rollback(completed_steps)
                        self.status = FailoverStatus.FAILED
                        return False
                    else:
                        logger.warning(f"Non-critical step failed: {step.name}: {e}")

            self.status = FailoverStatus.FAILOVER_COMPLETE
            self.failover_history.append({
                "action": "failover",
                "reason": reason,
                "timestamp": datetime.utcnow().isoformat(),
                "status": "success",
            })
            return True

        except Exception as e:
            logger.error(f"Failover failed with unexpected error: {e}")
            await self._rollback(completed_steps)
            self.status = FailoverStatus.FAILED
            return False

    def _get_failover_steps(self) -> List[FailoverStep]:
        """
        Define failover steps in dependency order.
        Databases first, then services, then routing.
        """
        return [
            # 1. Verify DR region is healthy
            FailoverStep(
                name="verify_dr_health",
                action=self._verify_dr_region_health,
                rollback=lambda: None,  # No rollback needed
                timeout_seconds=30.0,
                critical=True,
            ),
            # 2. Promote PostgreSQL replica to primary
            FailoverStep(
                name="promote_postgres",
                action=self._promote_postgres_replica,
                rollback=self._demote_postgres,
                timeout_seconds=120.0,
                critical=True,
            ),
            # 3. Activate vector DB replica
            FailoverStep(
                name="activate_vector_db",
                action=self._activate_vector_db_replica,
                rollback=self._deactivate_vector_db,
                timeout_seconds=60.0,
                critical=True,
            ),
            # 4. Warm up Redis cache
            FailoverStep(
                name="warm_redis_cache",
                action=self._warm_redis_cache,
                rollback=self._clear_redis_dr,
                timeout_seconds=300.0,
                critical=False,  # Non-critical: cache can warm up gradually
            ),
            # 5. Reconcile message broker queues
            FailoverStep(
                name="reconcile_queues",
                action=self._reconcile_message_queues,
                rollback=lambda: None,
                timeout_seconds=60.0,
                critical=True,
            ),
            # 6. Scale up DR services
            FailoverStep(
                name="scale_dr_services",
                action=self._scale_dr_services,
                rollback=self._scale_down_dr_services,
                timeout_seconds=120.0,
                critical=True,
            ),
            # 7. Update DNS routing
            FailoverStep(
                name="update_dns",
                action=self._update_dns_to_dr,
                rollback=self._update_dns_to_primary,
                timeout_seconds=60.0,
                critical=True,
            ),
            # 8. Verify DR region is serving traffic
            FailoverStep(
                name="verify_dr_serving",
                action=self._verify_dr_serving_traffic,
                rollback=lambda: None,
                timeout_seconds=30.0,
                critical=True,
            ),
        ]

    async def _verify_dr_region_health(self):
        """Verify that the DR region is healthy and ready."""
        import aiohttp
        async with aiohttp.ClientSession() as session:
            async with session.get(
                "https://dr.banking-genai.internal/health",
                timeout=aiohttp.ClientTimeout(total=10),
            ) as resp:
                if resp.status != 200:
                    raise HealthCheckError("DR region health check failed")
                data = await resp.json()
                if data.get("status") != "healthy":
                    raise HealthCheckError(f"DR region unhealthy: {data}")

    async def _promote_postgres_replica(self):
        """Promote the PostgreSQL replica in DR to primary."""
        import asyncpg
        conn = await asyncpg.connect(
            "postgresql://admin:password@dr-postgres.banking-genai.internal:5432/postgres"
        )
        await conn.execute("SELECT pg_promote()")
        await conn.close()
        logger.info("PostgreSQL DR replica promoted to primary")

    async def _activate_vector_db_replica(self):
        """Activate the vector DB replica for writes."""
        # In Qdrant, this means switching the DR node from read-only to read-write
        from qdrant_client import QdrantClient
        client = QdrantClient(url="https://dr-qdrant.banking-genai.internal:6333")
        # Trigger any write to verify write access
        client.http.put("/collections/_health_check", body={"test": True})
        logger.info("Vector DB DR replica activated for writes")

    async def _update_dns_to_dr(self):
        """Update DNS to route traffic to DR region."""
        import boto3
        route53 = boto3.client("route53")
        # Update the failover record to point to DR
        route53.change_resource_record_sets(
            HostedZoneId="Z1234567890",
            ChangeBatch={
                "Changes": [{
                    "Action": "UPSERT",
                    "ResourceRecordSet": {
                        "Name": "api.banking-genai.example.com",
                        "Type": "A",
                        "SetIdentifier": "dr-failover",
                        "Failover": "SECONDARY",
                        # In Route 53, failover records automatically
                        # route to SECONDARY when PRIMARY health check fails
                    },
                }]
            },
        )
        logger.info("DNS updated to route to DR region")

    async def _rollback(self, completed_steps: List[FailoverStep]):
        """Rollback completed steps in reverse order."""
        logger.warning("Rolling back failover")
        for step in reversed(completed_steps):
            try:
                await step.rollback()
                logger.info(f"Rolled back step: {step.name}")
            except Exception as e:
                logger.error(f"Rollback failed for {step.name}: {e}")
                # This is a critical scenario -- alert the team
                await send_alert("CRITICAL", f"Failover rollback failed: {step.name}: {e}")


class HealthCheckError(Exception):
    pass

async def send_alert(severity: str, message: str):
    """Send critical alert to on-call team."""
    # Implementation: PagerDuty, Slack, etc.
    pass
```

---

## LLM Provider Failover

```python
# dr/llm_failover.py
"""
LLM provider failover: when the primary LLM provider is unavailable,
automatically switch to the backup provider.
"""
import asyncio
from typing import Optional

class LLMFailoverHandler:
    """
    Handle LLM provider failures with automatic failover.
    Unlike infrastructure failover, this is per-request and transparent.
    """

    def __init__(self, providers: list):
        """
        providers: ordered list of provider configurations.
        First is primary, subsequent are failovers.
        """
        self.providers = providers
        self.health_status = {p["name"]: True for p in providers}
        self.circuit_breakers = {
            p["name"]: CircuitBreaker(failure_threshold=5, recovery_timeout=60)
            for p in providers
        }

    async def complete(self, prompt: str, **kwargs) -> dict:
        """
        Complete a prompt using the first healthy provider.
        Automatically fails over if the current provider fails.
        """
        last_error = None

        for provider in self.providers:
            name = provider["name"]

            # Check circuit breaker
            if not self.circuit_breakers[name].allow_request():
                continue

            try:
                result = await self._call_provider(provider, prompt, **kwargs)
                self.circuit_breakers[name].record_success()
                self.health_status[name] = True
                return result

            except Exception as e:
                self.circuit_breakers[name].record_failure()
                self.health_status[name] = False
                last_error = e
                logger.warning(f"LLM provider {name} failed: {e}")

        raise AllProvidersDownError(f"All LLM providers failed. Last error: {last_error}")

    async def _call_provider(self, provider: dict, prompt: str, **kwargs) -> dict:
        """Call a specific LLM provider."""
        import aiohttp
        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{provider['base_url']}/completions",
                json={"prompt": prompt, "model": provider["model"], **kwargs},
                headers={"Authorization": f"Bearer {provider['api_key']}"},
                timeout=aiohttp.ClientTimeout(total=provider.get("timeout", 30)),
            ) as resp:
                if resp.status != 200:
                    raise ProviderError(
                        f"Provider {provider['name']} returned {resp.status}: {await resp.text()}"
                    )
                return await resp.json()


class CircuitBreaker:
    """Circuit breaker pattern for service calls."""

    def __init__(self, failure_threshold: int = 5, recovery_timeout: int = 60):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failure_count = 0
        self.last_failure_time = 0
        self.state = "closed"  # closed, open, half-open

    def allow_request(self) -> bool:
        if self.state == "closed":
            return True
        if self.state == "open":
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = "half-open"
                return True
            return False
        return True  # half-open: allow one request

    def record_success(self):
        self.failure_count = 0
        self.state = "closed"

    def record_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = "open"


class AllProvidersDownError(Exception):
    pass

class ProviderError(Exception):
    pass
```

---

## DR Testing Schedule

| Test Type | Frequency | Scope | Duration | Disruption |
|---|---|---|---|---|
| Tabletop Exercise | Quarterly | All stakeholders | 2 hours | None |
| Component Failover | Monthly | Single component (DB, cache, etc.) | 1 hour | None (automated) |
| Region Failover | Semi-annually | Full region | 2-4 hours | Brief (during maintenance window) |
| LLM Provider Failover | Monthly | All LLM providers | 15 min | None (per-request) |
| Backup Restore | Quarterly | Audit logs, databases | 1 hour | None |
| Ransomware Recovery | Annually | Full system from backups | 4-8 hours | Full DR activation |

---

## Interview Questions

1. **What is the difference between RTO and RPO?**
   - RTO (Recovery Time Objective) is how long it takes to restore service after a disaster. RPO (Recovery Point Objective) is how much data you can afford to lose. RTO = time, RPO = data. In banking, RPO is often 0 for critical data (no data loss acceptable).

2. **How do you test DR without disrupting production?**
   - Component-level failover tests can run in production without impact (failover of a single pod, testing circuit breakers). Full region failover should be done during maintenance windows with traffic shifted first. Use traffic mirroring to validate the DR region handles real traffic correctly before actual failover.

3. **What is the hardest component to recover in a GenAI system?**
   - The vector database. It holds the indexed knowledge (embeddings) and can be terabytes in size. Cross-region replication is complex because vector databases were not originally designed for geo-replication. A full rebuild from the document store is possible but slow (hours to days).

4. **How do you handle LLM provider outages in a DR scenario?**
   - LLM provider failover is independent of infrastructure DR. Use circuit breakers and multiple provider configurations. If all providers are down (rare but possible during internet-scale outages), fall back to cached responses, reduced functionality, or a graceful degradation message.

---

## Cross-References

- See [architecture/multi-region-design.md](./multi-region-design.md) for multi-region architecture
- See [infrastructure/backup-restore.md](../infrastructure/backup-restore.md) for backup strategies
- See [kubernetes-openshift/disaster-recovery.md](../kubernetes-openshift/disaster-recovery.md) for K8s DR
- See [incident-management/](../incident-management/) for incident response procedures
