# OpenShift Operators and SDK

## Overview

Operators automate the management of complex applications on Kubernetes. This guide covers using existing Operators and building custom Operators for banking GenAI platforms.

## Using Operators

```yaml
# Install PostgreSQL Operator via OperatorHub
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: CrunchyPostgres
  namespace: openshift-operators
spec:
  channel: v5
  name: crunchy-postgres
  source: operatorhubio-catalog
  sourceNamespace: olm

# Create PostgreSQL cluster
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: banking-postgres
  namespace: banking-data
spec:
  postgresVersion: 15
  instances:
    - name: instance1
      replicas: 2
      dataVolumeClaimSpec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 100Gi
  backups:
    pgbackrest:
      repos:
        - name: repo1
          volume:
            volumeClaimSpec:
              accessModes:
                - ReadWriteOnce
              resources:
                requests:
                  storage: 50Gi
```

## Custom Operator (SDK)

```go
// Simple Go-based Operator for GenAI document processing
package controllers

import (
    "context"
    genai "github.com/bank/genai-operator/api/v1alpha1"
    appsv1 "k8s.io/api/apps/v1"
    ctrl "sigs.k8s.io/controller-runtime"
)

type GenAIServiceReconciler struct {
    ctrl.Reconciler
}

func (r *GenAIServiceReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // Fetch GenAIService instance
    service := &genai.GenAIService{}
    if err := r.Get(ctx, req.NamespacedName, service); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // Deploy managed resources
    deployment := r.deploymentForGenAI(service)
    if err := ctrl.SetControllerReference(service, deployment, r.Scheme); err != nil {
        return ctrl.Result{}, err
    }
    
    // Create or update
    found := &appsv1.Deployment{}
    err := r.Get(ctx, types.NamespacedName{Name: deployment.Name, Namespace: deployment.Namespace}, found)
    if err != nil {
        return ctrl.Result{}, r.Create(ctx, deployment)
    }
    
    return ctrl.Result{}, nil
}
```

## Cross-References

- **StatefulSets**: See [statefulsets.md](statefulsets.md) for stateful workloads
- **Helm**: See [helm.md](helm.md) for application packaging

## Interview Questions

1. **What is a Kubernetes Operator? What problem does it solve?**
2. **When would you build a custom Operator vs using Helm charts?**
3. **How does the Operator SDK work?**
4. **What are common Operators used in banking environments?**
5. **How do Operators handle upgrades and backups?**
6. **What is the Operator Lifecycle Manager (OLM)?**

## Checklist: Operator Usage

- [ ] OperatorHub used for common software (databases, caches)
- [ ] Custom Operators built only when existing solutions don't fit
- [ ] Operator CRDs versioned and documented
- [ ] Backup handling built into Operators
- [ ] Operator upgrades tested before production
- [ ] Operator resource limits configured
- [ ] Operator logging monitored
