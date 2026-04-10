# StatefulSets for Databases and Message Queues

## Overview

StatefulSets manage stateful applications in Kubernetes, providing stable network identities, persistent storage, and ordered deployment. This guide covers running databases and message queues on Kubernetes for banking platforms.

## StatefulSet Fundamentals

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
  namespace: banking-data
spec:
  serviceName: postgresql-headless
  replicas: 3
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
        - name: postgresql
          image: postgres:15
          ports:
            - containerPort: 5432
              name: postgres
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: password
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              cpu: "1"
              memory: 2Gi
            limits:
              cpu: "2"
              memory: 4Gi
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3
        resources:
          requests:
            storage: 100Gi
---
# Headless service for StatefulSet DNS
apiVersion: v1
kind: Service
metadata:
  name: postgresql-headless
  namespace: banking-data
spec:
  clusterIP: None
  selector:
    app: postgresql
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
```

## Stable Network Identity

```
StatefulSet pod naming:
postgresql-0.postgresql-headless.banking-data.svc.cluster.local
postgresql-1.postgresql-headless.banking-data.svc.cluster.local
postgresql-2.postgresql-headless.banking-data.svc.cluster.local

Each pod gets:
- Stable hostname (ordinal-based)
- Stable persistent volume (survives pod restart)
- Ordered deployment (0 -> 1 -> 2)
- Ordered termination (reverse order)
```

## Cross-References

- **Helm**: See [helm.md](helm.md) for packaging StatefulSets
- **Storage**: See [statefulsets.md](statefulsets.md) for storage classes

## Interview Questions

1. **What is the difference between a Deployment and a StatefulSet?**
2. **How do StatefulSets provide stable network identities?**
3. **When would you run a database in Kubernetes vs managed service?**
4. **How do persistent volumes work with StatefulSets?**
5. **What is a headless service? Why do StatefulSets need it?**
6. **How do you backup a StatefulSet-based database?**

## Checklist: StatefulSet Best Practices

- [ ] Headless service configured for DNS discovery
- [ ] Persistent volume claims with appropriate storage class
- [ ] Resource requests and limits set
- [ ] Pod disruption budget for minimum availability
- [ ] Backup procedure documented for persistent volumes
- [ ] Ordered deployment understood and tested
- [ ] Monitoring configured for stateful workloads
- [ ] Consider managed service (RDS) for production databases
