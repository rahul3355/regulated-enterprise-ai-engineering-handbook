# Debugging Kubernetes Pods

## Overview

Debugging pods in production requires understanding container internals, K8s networking, and available debugging tools. This guide covers practical debugging techniques for banking GenAI applications.

## Debugging Techniques

```bash
# 1. Check pod status and events
kubectl get pods -n banking-genai -o wide
kubectl describe pod <pod-name> -n banking-genai

# 2. View logs
kubectl logs <pod-name> -n banking-genai
kubectl logs <pod-name> -n banking-genai --previous  # Previous crash
kubectl logs <pod-name> -n banking-genai -c <container>  # Multi-container

# 3. Execute commands in running pod
kubectl exec -it <pod-name> -n banking-genai -- /bin/sh
kubectl exec <pod-name> -n banking-genai -- ps aux
kubectl exec <pod-name> -n banking-genai -- env

# 4. Port forwarding for local debugging
kubectl port-forward <pod-name> 8080:8080 -n banking-genai

# 5. Check resource usage
kubectl top pod <pod-name> -n banking-genai

# 6. Ephemeral containers (K8s 1.23+)
kubectl debug -it <pod-name> -n banking-genai --image=busybox --target=<container>

# 7. Check network connectivity
kubectl exec <pod-name> -n banking-genai -- nslookup genai-api.banking-genai
kubectl exec <pod-name> -n banking-genai -- curl -v http://postgres.banking-data:5432

# 8. Check node conditions
kubectl describe node <node-name>
kubectl get events -n banking-genai --sort-by='.lastTimestamp'
```

## Common Issues

| Symptom | Likely Cause | Debug Command |
|---------|-------------|---------------|
| Pending | Insufficient resources | `kubectl describe pod` |
| CrashLoopBackOff | App crash | `kubectl logs --previous` |
| ImagePullBackOff | Image/auth issue | `kubectl describe pod` |
| OOMKilled | Memory limit exceeded | `kubectl describe pod` |
| Not Ready | Probe failing | `kubectl describe pod` |
| DNS failure | CoreDNS issue | `kubectl exec -- nslookup` |

## Cross-References

- **Pods**: See [pods.md](pods.md) for pod lifecycle
- **Production Incidents**: See [production-incidents.md](production-incidents.md) for real scenarios

## Interview Questions

1. **A pod is in CrashLoopBackOff. Walk me through your debugging process.**
2. **How do you debug network connectivity issues between pods?**
3. **What are ephemeral containers? When do you use them?**
4. **How do you debug a pod that's stuck in Pending?**
5. **What tools do you use for production debugging in Kubernetes?**
6. **How do you debug high memory usage in a running pod?**

## Checklist: Debugging Readiness

- [ ] Container images include debugging tools (curl, nslookup, ps)
- [ ] Health probes configured and tested
- [ ] Resource requests/limits set appropriately
- [ ] Application logs written to stdout/stderr
- [ ] Log levels configurable at runtime
- [ ] Ephemeral debugging container images available
- [ ] Port forwarding tested for local debugging
- [ ] Node monitoring enabled
- [ ] Events logged and monitored
