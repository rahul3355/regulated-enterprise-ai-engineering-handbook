# Kubernetes Secrets: Management, Encryption, and Rotation

## Overview

Secrets store sensitive data like passwords, API keys, and certificates. In banking, secret management is critical for security compliance. This guide covers secret creation, encryption at rest, rotation strategies, and integration with external secret managers.

## Secret Fundamentals

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: banking-genai
type: Opaque
data:
  # Values must be base64 encoded
  username: YmFua2luZ191c2Vy  # banking_user
  password: c2VjdXJlUGFzc3dvcmQxMjM=  # securePassword123
  url: cG9zdGdyZXNxbDovL2JhbmtpbmdfdXNlcjpzZWN1cmVQYXNzd29yZDEyM0Bwb3N0Z3Jlcy1wcm9kOjU0MzIvYmFua2luZw==
---
# StringData: values are automatically base64 encoded
apiVersion: v1
kind: Secret
metadata:
  name: openai-api-key
  namespace: banking-genai
type: Opaque
stringData:
  api-key: "sk-proj-xxxxx"
  org-id: "org-xxxxx"
```

## Using Secrets

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: genai-api
spec:
  template:
    spec:
      containers:
        - name: api
          image: quay.io/banking/genai-api:1.0.0
          # 1. Environment variable from secret
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: url
            - name: OPENAI_API_KEY
              valueFrom:
                secretKeyRef:
                  name: openai-api-key
                  key: api-key
          
          # 2. Volume mount
          volumeMounts:
            - name: secret-volume
              mountPath: /app/secrets
              readOnly: true
      volumes:
        - name: secret-volume
          secret:
            secretName: db-credentials
            defaultMode: 0400  # Read-only for owner
```

## External Secrets Manager Integration

```yaml
# Using External Secrets Operator with HashiCorp Vault
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: banking-genai
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
    - secretKey: username
      remoteRef:
        key: banking/genai/database
        property: username
    - secretKey: password
      remoteRef:
        key: banking/genai/database
        property: password
    - secretKey: url
      remoteRef:
        key: banking/genai/database
        property: url
---
# ClusterSecretStore for Vault
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault.bank.com:8200"
      path: "secret"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "genai-role"
```

## Secret Rotation

```yaml
# Automated rotation with external-secrets
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: rotating-db-password
  namespace: banking-genai
spec:
  refreshInterval: 24h  # Check for new secret every 24 hours
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: db-credentials
    template:
      engineVersion: v2
      data:
        password: "{{ .password }}"
        url: "postgresql://{{ .username }}:{{ .password }}@{{ .host }}:{{ .port }}/{{ .database }}"
  data:
    - secretKey: password
      remoteRef:
        key: banking/genai/database
        property: password
    - secretKey: username
      remoteRef:
        key: banking/genai/database
        property: username
    - secretKey: host
      remoteRef:
        key: banking/genai/database
        property: host
```

## Encryption at Rest

```yaml
# Enable encryption in K8s (EncryptionConfiguration)
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-key>
      - identity: {}  # Fallback (unencrypted)
```

## Cross-References

- **ConfigMaps**: See [configmaps.md](configmaps.md) for non-sensitive configuration
- **RBAC**: See [rbac.md](rbac.md) for access control

## Interview Questions

1. **How are Kubernetes secrets stored? Are they encrypted by default?**
2. **How do you integrate Kubernetes with HashiCorp Vault?**
3. **What is secret rotation and why is it important?**
4. **How do you enable encryption at rest for Kubernetes secrets?**
5. **What are the security risks of storing secrets in Kubernetes?**
6. **How does External Secrets Operator work?**

## Checklist: Secret Management

- [ ] Encryption at rest enabled for etcd
- [ ] Secrets not stored in version control
- [ ] External secret manager used for production (Vault, AWS Secrets Manager)
- [ ] Secret rotation automated
- [ ] Secret access audited (RBAC)
- [ ] Secrets mounted as volumes with restrictive permissions (0400)
- [ ] Secret names and keys follow naming convention
- [ ] No secrets in environment variable defaults
- [ ] Audit logging for secret access enabled
