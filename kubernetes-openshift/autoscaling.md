# Kubernetes Autoscaling: HPA, VPA, and KEDA

## Overview

Autoscaling ensures GenAI applications handle variable loads efficiently. This guide covers Horizontal Pod Autoscaler (HPA), Vertical Pod Autoscaler (VPA), and Kubernetes Event-Driven Autoscaling (KEDA) for banking workloads.

## Horizontal Pod Autoscaler (HPA)

```yaml
# Scale based on CPU/memory utilization
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: genai-api-hpa
  namespace: banking-genai
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: genai-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
    # CPU-based scaling
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    # Memory-based scaling
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    # Custom metric (requests per second)
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
        - type: Percent
          value: 50
          periodSeconds: 60
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120
      selectPolicy: Min
```

## KEDA: Event-Driven Autoscaling

```yaml
# Scale based on Kafka queue depth
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: document-processor-scaling
  namespace: banking-genai
spec:
  scaleTargetRef:
    name: document-processor
  minReplicaCount: 1
  maxReplicaCount: 10
  pollingInterval: 15
  cooldownPeriod: 120
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka.banking-data:9092
        consumerGroup: document-processor
        topic: banking-documents
        lagThreshold: "100"
        offsetResetPolicy: latest
---
# Scale based on Prometheus metrics
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: embedding-service-scaling
spec:
  scaleTargetRef:
    name: embedding-service
  minReplicaCount: 2
  maxReplicaCount: 15
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.monitoring:9090
        metricName: embedding_queue_depth
        threshold: "50"
        query: |
          sum(embedding_queue_depth{service="embedding-service"})
```

## Vertical Pod Autoscaler (VPA)

```yaml
# Recommend or set resource requests/limits
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutimizer
metadata:
  name: genai-api-vpa
  namespace: banking-genai
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: genai-api
  updatePolicy:
    updateMode: "Auto"  # Off, Initial, Auto
  resourcePolicy:
    containerPolicies:
      - containerName: api
        minAllowed:
          cpu: 100m
          memory: 256Mi
        maxAllowed:
          cpu: "2"
          memory: 4Gi
```

## Cross-References

- **GPU Workloads**: See [gpu-workloads.md](gpu-workloads.md) for GPU autoscaling
- **Deployments**: See [deployments.md](deployments.md) for deployment management

## Interview Questions

1. **How does HPA work? What metrics can trigger scaling?**
2. **What is the difference between HPA and VPA? When do you use each?**
3. **How does KEDA enable event-driven autoscaling?**
4. **Your GenAI API needs to scale based on queue depth. How do you configure it?**
5. **What are the risks of aggressive autoscaling?**
6. **How do you set appropriate min/max replicas for a banking API?**

## Checklist: Autoscaling

- [ ] HPA configured with appropriate CPU/memory targets
- [ ] minReplicas set for baseline availability
- [ ] maxReplicas set to prevent runaway scaling
- [ ] Scale-down stabilization window configured
- [ ] KEDA installed for event-driven scaling needs
- [ ] VPA in recommendation mode before auto mode
- [ ] Resource requests accurately set (not too high, not too low)
- [ ] Cluster capacity monitored (node utilization)
- [ ] Cluster autoscaler configured for node scaling
- [ ] Scaling behavior tested with load tests
