# TLS and Certificates

## Overview

Transport Layer Security (TLS) is the foundation of secure communication in banking systems. It provides confidentiality (encryption), integrity (tamper detection), and authentication (identity verification) for all network traffic. This guide covers TLS versions, cipher suites, certificate management, and mutual TLS (mTLS) for service-to-service communication in Kubernetes and OpenShift.

## TLS Protocol Versions

| Version | Year | Status | Banking Recommendation |
|---|---|---|---|
| SSL 2.0 | 1995 | CRITICAL VULNERABILITIES | NEVER |
| SSL 3.0 | 1996 | POODLE attack (CVE-2014-3566) | NEVER |
| TLS 1.0 | 1999 | Deprecated (RFC 8996) | NEVER |
| TLS 1.1 | 2006 | Deprecated (RFC 8996) | NEVER |
| TLS 1.2 | 2008 | Current minimum | REQUIRED MINIMUM |
| TLS 1.3 | 2018 | Recommended | PREFERRED |

### Real-World TLS Failures

- **Heartbleed (CVE-2014-0160)**: OpenSSL vulnerability allowed reading 64KB of server memory. Affected 17% of all secure websites. Exposed private keys, passwords, and session tokens.
- **POODLE (CVE-2014-3566)**: SSL 3.0 downgrade attack. Forced deprecation of SSL 3.0.
- **ROBOT (2017)**: Return Of Bleichenbacher's Oracle Threat. Affected RSA key exchange in TLS, allowing decryption of traffic.
- **Logjam (CVE-2015-4000)**: Downgrade attack on Diffie-Hellman key exchange. 8.4% of Alexa Top 1M sites affected.

## Cipher Suites

### TLS 1.3 Cipher Suites (Recommended)

TLS 1.3 simplified cipher suites to only secure options:

```
TLS_AES_256_GCM_SHA384          # Recommended
TLS_CHACHA20_POLY1305_SHA256    # Good for mobile (software-optimized)
TLS_AES_128_GCM_SHA256          # Acceptable
```

### TLS 1.2 Cipher Suites (Acceptable)

```
# Strong cipher suites (in order of preference)
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
TLS_DHE_RSA_WITH_AES_256_GCM_SHA384
```

### Insecure Cipher Suites (Block These)

```
# NULL ciphers (no encryption)
TLS_NULL_WITH_NULL_NULL

# Export-grade ciphers (40/56-bit, deliberately weakened)
TLS_RSA_EXPORT_WITH_RC4_40_MD5

# RC4 (biased keystream, practical attacks)
TLS_RSA_WITH_RC4_128_SHA

# DES/3DES (Sweet32 attack, CVE-2016-2183)
TLS_RSA_WITH_3DES_EDE_CBC_SHA

# CBC mode ciphers (BEAST, Lucky13 attacks)
TLS_RSA_WITH_AES_128_CBC_SHA

# Static RSA (no forward secrecy)
TLS_RSA_WITH_AES_256_GCM_SHA384
```

### Server TLS Configuration

```python
# Python (FastAPI/Uvicorn)
import ssl

ssl_context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
ssl_context.minimum_version = ssl.TLSVersion.TLSv1_2
ssl_context.set_ciphers(
    'ECDHE+AESGCM:ECDHE+CHACHA20:DHE+AESGCD:!aNULL:!MD5:!DSS:!RC4:!3DES'
)
ssl_context.options |= ssl.OP_NO_COMPRESSION  # CRIME attack
ssl_context.options |= ssl.OP_NO_RENEGOTIATION

# Nginx configuration
server {
    listen 443 ssl http2;
    server_name api.bank.example.com;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305';
    ssl_prefer_server_ciphers on;

    ssl_certificate /etc/tls/tls.crt;
    ssl_certificate_key /etc/tls/tls.key;

    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/tls/ca.crt;
}
```

## Certificate Management

### Certificate Anatomy

```
X.509 Certificate:
├── Version (v3)
├── Serial Number
├── Signature Algorithm (SHA256withRSA)
├── Issuer (CA that signed it)
├── Validity Period
│   ├── Not Before: 2024-01-01
│   └── Not After: 2024-04-01  (90-day max for public certs)
├── Subject (entity the cert belongs to)
│   └── CN = api.bank.example.com
├── Subject Public Key Info
│   └── RSA 2048-bit / ECDSA P-256
├── Extensions
│   ├── Subject Alternative Names (SANs)
│   │   ├── DNS: api.bank.example.com
│   │   └── DNS: api-v2.bank.example.com
│   ├── Key Usage
│   │   └── Digital Signature, Key Encipherment
│   ├── Extended Key Usage
│   │   └── TLS Web Server Authentication
│   └── Basic Constraints
│       └── CA: FALSE
└── Signature (from CA)
```

### Certificate Chain

```mermaid
graph TD
    A[Root CA] -->|Signs| B[Intermediate CA]
    B -->|Signs| C[Leaf Certificate]
    C -->|Served by| D[api.bank.example.com]

    style A fill:#f96,stroke:#333
    style B fill:#9cf,stroke:#333
    style C fill:#9f9,stroke:#333
```

The server must provide the full chain (leaf + intermediates), NOT just the leaf certificate. Clients already have root CAs in their trust store.

### Certificate Generation with cert-manager (Kubernetes)

```yaml
# cert-manager Issuer (namespace-scoped)
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-prod
  namespace: production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ssl-admin@bank.example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - http01:
        ingress:
          class: nginx

---
# Certificate resource
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: api-bank-tls
  namespace: production
spec:
  secretName: api-bank-tls-secret
  duration: 2160h     # 90 days
  renewBefore: 360h   # Renew 15 days before expiry
  issuerRef:
    name: letsencrypt-prod
    kind: Issuer
  commonName: api.bank.example.com
  dnsNames:
  - api.bank.example.com
  - api-v2.bank.example.com
  usages:
  - digital signature
  - key encipherment
  - server auth
  privateKey:
    algorithm: RSA
    encoding: PKCS8
    size: 2048
  # For internal services, use a private CA instead of Let's Encrypt
```

### Internal CA with cert-manager

```yaml
# Self-signed root CA
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: bank-root-ca
spec:
  selfSigned: {}

---
# Root CA certificate
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: bank-root-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: "Bank Root CA"
  secretName: bank-root-ca-secret
  privateKey:
    algorithm: ECDSA
    size: 384
  issuerRef:
    name: bank-root-ca
    kind: ClusterIssuer

---
# Intermediate CA
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: bank-intermediate-ca
spec:
  ca:
    secretName: bank-intermediate-ca-secret

---
# Intermediate CA certificate
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: bank-intermediate-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: "Bank Intermediate CA"
  secretName: bank-intermediate-ca-secret
  issuerRef:
    name: bank-root-ca
    kind: ClusterIssuer
    group: cert-manager.io

---
# Issuer for banking services
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: bank-services-issuer
spec:
  ca:
    secretName: bank-intermediate-ca-secret
```

## Mutual TLS (mTLS)

### mTLS Architecture

```mermaid
sequenceDiagram
    participant Client as Client Service
    participant Server as Server Service
    participant CA as Internal CA

    Note over Client,CA: Both services have certificates<br/>signed by the same CA

    Client->>Server: TLS ClientHello
    Server->>Client: Server Certificate
    Client->>Client: Verify server cert against CA
    Server->>Client: CertificateRequest (ask for client cert)
    Client->>Server: Client Certificate
    Server->>Server: Verify client cert against CA
    Server->>Client: ServerHelloDone
    Client->>Server: Finished (encrypted)
    Server->>Client: Finished (encrypted)

    Note over Client,Server: mTLS established.<br/>Both parties authenticated.
```

### mTLS with Istio Service Mesh

```yaml
# Enable mTLS mesh-wide
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT  # All traffic must be mTLS

---
# Per-service mTLS policy
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: banking-api-mtls
  namespace: production
spec:
  selector:
    matchLabels:
      app: banking-api
  mtls:
    mode: STRICT

---
# Allow plaintext for health checks (if needed)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: banking-api-permissive
  namespace: production
spec:
  selector:
    matchLabels:
      app: banking-api
  mtls:
    mode: PERMISSIVE  # Accept both mTLS and plaintext (migration only)

---
# Authorization: Only specific services can access banking-api
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: banking-api-access
  namespace: production
spec:
  selector:
    matchLabels:
      app: banking-api
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/production/sa/transaction-service"
        - "cluster.local/ns/production/sa/notification-service"
    to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/api/*"]
```

### mTLS Certificate Rotation

```yaml
# Istio Citadel handles automatic certificate rotation
# Default rotation: every 24 hours
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      # Certificate rotation
      mtls:
        enabled: true
      # Workload certificate lifetime
      pilotCertProvider: istiod
  meshConfig:
    defaultConfig:
      # Rotation interval
      holdApplicationUntilProxyStarts: true
```

## Certificate Monitoring

```python
# Monitor certificate expiry across all services
import ssl
import socket
from datetime import datetime, timedelta
from typing import Optional

def check_certificate_expiry(hostname: str, port: int = 443) -> dict:
    """Check TLS certificate expiry for a service"""
    context = ssl.create_default_context()
    with socket.create_connection((hostname, port), timeout=10) as sock:
        with context.wrap_socket(sock, server_hostname=hostname) as ssock:
            cert = ssock.getpeercert()
            not_after = datetime.strptime(cert['notAfter'], '%b %d %H:%M:%S %Y %Z')
            days_remaining = (not_after - datetime.utcnow()).days

            return {
                "hostname": hostname,
                "issuer": cert['issuer'],
                "not_after": not_after.isoformat(),
                "days_remaining": days_remaining,
                "status": "OK" if days_remaining > 30 else "WARNING" if days_remaining > 7 else "CRITICAL",
            }

# Monitor all banking services
services = [
    ("api.bank.example.com", 443),
    ("auth.bank.example.com", 443),
    ("transactions.bank.example.com", 443),
    ("internal-api.bank.svc.cluster.local", 8443),
]

alerts = []
for hostname, port in services:
    result = check_certificate_expiry(hostname, port)
    if result["status"] in ("WARNING", "CRITICAL"):
        alerts.append(result)
        send_alert(f"Certificate expiring for {hostname} in {result['days_remaining']} days")
```

## Banking-Specific TLS Requirements

### PCI-DSS TLS Requirements

| Requirement | Description | Implementation |
|---|---|---|
| 2.3 | Encrypt all non-console administrative access | SSH, TLS for all admin interfaces |
| 4.1 | Use strong cryptography for cardholder data in transit | TLS 1.2+ with approved ciphers |
| 4.2 | Never send PANs via unencrypted channels | DLP, TLS enforcement |

### TLS Configuration Standards

```yaml
# Minimum TLS configuration for banking services
tls_standards:
  minimum_version: TLSv1.2
  preferred_version: TLSv1.3

  # Approved cipher suites (TLS 1.2)
  cipher_suites_1_2:
    - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
    - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
    - TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
    - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256

  # TLS 1.3 cipher suites (all are secure)
  cipher_suites_1_3:
    - TLS_AES_256_GCM_SHA384
    - TLS_CHACHA20_POLY1305_SHA256
    - TLS_AES_128_GCM_SHA256

  # Key exchange (must provide forward secrecy)
  key_exchange:
    - ECDHE
    - DHE

  # Signature algorithms
  signature_algorithms:
    - rsa_pss_rsae_sha256
    - ecdsa_secp256r1_sha256
    - ed25519

  # Certificate requirements
  certificates:
    minimum_key_size_rsa: 2048
    preferred_key_type: ECDSA-P256
    maximum_validity_days: 90
    require_san: true
    require_ocsp_stapling: true
```

## OpenShift-Specific TLS Configuration

### OpenShift Router TLS

```yaml
# Configure OpenShift router with custom TLS settings
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: default
  namespace: openshift-ingress-operator
spec:
  defaultCertificate:
    name: banking-wildcard-tls
  clientTLS:
    clientCA:
      name: bank-ca-bundle
    clientCertificatePolicy: Require  # Require client certificates
  tlsSecurityProfile:
    type: Custom
    custom:
      ciphers:
        - ECDHE-ECDSA-AES256-GCM-SHA384
        - ECDHE-RSA-AES256-GCM-SHA384
        - ECDHE-ECDSA-CHACHA20-POLY1305
        - ECDHE-RSA-CHACHA20-POLY1305
      minTLSVersion: VersionTLS12
```

### Route with mTLS

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: banking-api
  namespace: production
spec:
  host: api.bank.example.com
  to:
    kind: Service
    name: banking-api
  port:
    targetPort: 8443
  tls:
    termination: reencrypt          # TLS from client, re-encrypt to backend
    insecureEdgeTerminationPolicy: Redirect  # Redirect HTTP to HTTPS
    destinationCACertificate: |    # CA cert for backend verification
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
    clientCertificate: |           # Client cert for backend
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
    key: |                         # Client key for backend
      -----BEGIN RSA PRIVATE KEY-----
      ...
      -----END RSA PRIVATE KEY-----
```

## Security Testing

### TLS Testing with testssl.sh

```bash
# Comprehensive TLS testing
testssl.sh --full api.bank.example.com

# Check for specific vulnerabilities
testssl.sh --heartbleed api.bank.example.com
testssl.sh --ccs api.bank.example.com
testssl.sh --robot api.bank.example.com

# Check certificate chain
testssl.sh --server-defaults api.bank.example.com

# CI/CD integration
tls_scan_results=$(testssl.sh --jsonfile api.bank.example.com)
if jq '.vulnerabilities | any(.severity == "HIGH" or .severity == "CRITICAL")' $tls_scan_results; then
  echo "TLS scan failed - blocking deployment"
  exit 1
fi
```

### Automated TLS Monitoring

```yaml
# Prometheus alert for certificate expiry
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: certificate-expiry
  namespace: monitoring
spec:
  groups:
  - name: certificates
    rules:
    - alert: CertificateExpiryWarning
      expr: cert_manager_certificate_expiration_timestamp_seconds - time() < 86400 * 30
      for: 1h
      labels:
        severity: warning
      annotations:
        summary: "TLS certificate expiring in {{ $value | humanizeDuration }}"
    - alert: CertificateExpiryCritical
      expr: cert_manager_certificate_expiration_timestamp_seconds - time() < 86400 * 7
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "TLS certificate expiring in {{ $value | humanizeDuration }}"
```

## Interview Questions

### Junior Level

1. What is the difference between TLS 1.2 and TLS 1.3?
2. What does "forward secrecy" mean and why is it important?
3. What is a certificate chain?
4. What is mTLS and when would you use it?

### Senior Level

1. How do you implement automatic certificate rotation in Kubernetes?
2. What is OCSP stapling and why is it needed?
3. Explain how you would debug an mTLS connection failure between two services.
4. Why should you never use TLS 1.0 or 1.1?

### Staff Level

1. Design a certificate management strategy for 500 services across 5 environments.
2. How would you detect and respond to a compromised internal CA?
3. What is your strategy for managing TLS configuration drift across hundreds of services?

## Cross-References

- [Network Security](./network-security.md) - Network-level encryption requirements
- [Service-to-Service Security](./service-to-service-security.md) - mTLS between services
- [Kubernetes Security](./kubernetes-security.md) - K8s certificate management
- [Encryption](./encryption.md) - Encryption fundamentals
