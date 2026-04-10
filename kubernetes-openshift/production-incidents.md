# Production Kubernetes Incidents and Lessons

## Overview

Learning from production incidents is essential for building reliable banking GenAI platforms. This guide documents common Kubernetes incident patterns, root causes, and preventive measures.

## Incident Patterns

### Incident 1: Pod OOMKilled During Traffic Spike

```
Timeline:
14:00 - Marketing campaign drives 3x traffic increase
14:05 - GenAI API pods start hitting memory limits
14:07 - Pods OOMKilled, HPA scaling up
14:10 - New pods also OOMKilled (memory limit too low)
14:15 - Service degraded, 50% of requests failing
14:20 - Emergency: Increase memory limits
14:30 - Service recovered

Root Cause: Memory limit set too low for traffic spike
Impact: 30 minutes of degraded service
Fix: Increased memory limit from 1Gi to 2Gi, added load testing
```

### Incident 2: Etcd Cluster Failure

```
Timeline:
03:00 - etcd disk usage reaches 95%
03:05 - etcd leader election fails
03:07 - API server becomes unresponsive
03:10 - All pod scheduling stops
03:15 - On-call alerted
03:30 - etcd disk expanded, cluster recovered

Root Cause: Insufficient etcd disk space, no alerting
Impact: 25 minutes of no new deployments or scaling
Fix: Increased etcd disk, added disk usage alerts
```

### Incident 3: Network Policy Misconfiguration

```
Timeline:
10:00 - Network policy updated to restrict egress
10:01 - GenAI API can no longer reach PostgreSQL
10:05 - Customer-facing errors increase
10:10 - Policy rollback
10:15 - Service recovered

Root Cause: Egress policy missing DNS and database rules
Impact: 15 minutes of API failures
Fix: Added DNS and database egress rules, policy testing in CI
```

## Prevention Checklist

- [ ] Load testing performed before major releases
- [ ] Resource limits reviewed quarterly
- [ ] etcd monitoring and alerting enabled
- [ ] Network policy changes tested in staging
- [ ] Pod disruption budgets configured
- [ ] Cluster capacity planning reviewed monthly
- [ ] Incident runbooks updated
- [ ] Post-mortems conducted for all P1/P2 incidents
- [ ] Chaos engineering tests run regularly
- [ ] Dependency health checks enabled

## Cross-References

- **Debugging**: See [debugging-pods.md](debugging-pods.md) for debugging techniques
- **Disaster Recovery**: See [disaster-recovery.md](../databases/disaster-recovery.md) for DR

## Interview Questions

1. **Describe a production Kubernetes incident you handled. What did you learn?**
2. **How do you prevent OOMKilled events during traffic spikes?**
3. **What happens when etcd fails? How do you recover?**
4. **How do you test network policies before applying them?**
5. **What is your incident response process for Kubernetes?**
6. **How do you conduct post-mortems for Kubernetes incidents?**

## Checklist: Incident Preparedness

- [ ] Runbooks for common failure scenarios
- [ ] On-call rotation with escalation paths
- [ ] Monitoring dashboards for cluster health
- [ ] Alerting thresholds tuned (not too noisy)
- [ ] Incident communication templates
- [ ] Post-mortem process documented
- [ ] Action items tracked and completed
- [ ] Chaos engineering tests scheduled
