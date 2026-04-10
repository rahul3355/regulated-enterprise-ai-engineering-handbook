# OpenShift Routes vs Ingress

## Overview

OpenShift Routes provide external access to services, serving a similar purpose to Kubernetes Ingress but with OpenShift-specific features. This guide compares Routes and Ingress, covering when to use each in banking environments.

## Route Configuration

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: genai-api
  namespace: banking-genai
  annotations:
    haproxy.router.openshift.io/timeout: 120s
    haproxy.router.openshift.io/rate-limit-requests: "100"
    haproxy.router.openshift.io/rate-limit-seconds: "60"
spec:
  host: genai-api.bank.com
  path: /api/v1
  to:
    kind: Service
    name: genai-api
    weight: 100
  port:
    targetPort: 8080
  tls:
    termination: edge  # TLS terminated at router
    insecureEdgeTerminationPolicy: Redirect  # HTTP -> HTTPS
---
# Passthrough TLS (TLS to backend)
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: genai-api-passthrough
spec:
  host: genai-api.bank.com
  to:
    kind: Service
    name: genai-api
  port:
    targetPort: 8443
  tls:
    termination: passthrough
---
# Re-encrypt TLS (router terminates, re-encrypts to backend)
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: genai-api-reencrypt
spec:
  host: genai-api.bank.com
  to:
    kind: Service
    name: genai-api
  tls:
    termination: reencrypt
    destinationCACertificate: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
```

## Route vs Ingress Comparison

| Feature | OpenShift Route | K8s Ingress |
|---------|----------------|-------------|
| TLS Termination | edge, passthrough, reencrypt | Standard TLS |
| Load Balancing | Weighted (for A/B, canary) | Via annotations |
| Rate Limiting | Built-in annotations | Via controller |
| WebSocket | Supported | Supported |
| Multiple Paths | Single path per route | Multiple paths |
| Wildcard Routes | Supported | Controller-dependent |
| Router | HAProxy (default) | Controller-dependent |

## Cross-References

- **Ingress**: See [ingress.md](ingress.md) for standard K8s ingress
- **Deployments**: See [deployments.md](deployments.md) for deployment strategies

## Interview Questions

1. **What is the difference between OpenShift Routes and Kubernetes Ingress?**
2. **What are the three TLS termination modes in OpenShift Routes?**
3. **How do you implement canary deployments with OpenShift Routes?**
4. **When would you choose Route over Ingress in an OpenShift cluster?**
5. **How do you configure rate limiting on an OpenShift Route?**
6. **What is HAProxy router in OpenShift?**

## Checklist: Route Configuration

- [ ] TLS termination mode chosen appropriately
- [ ] HTTP-to-HTTPS redirect enabled
- [ ] Rate limiting configured
- [ ] Timeouts set for GenAI workloads
- [ ] Route host matches DNS records
- [ ] Health check endpoint accessible
- [ ] Route monitoring enabled
