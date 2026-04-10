# Secrets Management

## Overview

Secrets management is the practice of securely storing, distributing, rotating, and revoking sensitive credentials: API keys, database passwords, TLS certificates, encryption keys, and tokens. A single leaked secret can compromise an entire banking platform. This guide covers production-grade secrets management using HashiCorp Vault, Kubernetes native secrets, and automated rotation.

## Threat Landscape

### How Secrets Get Leaked

| Vector | Real-World Example | Impact |
|---|---|---|
| Committed to Git | Uber (2016): AWS keys in GitHub exposed 57M user records | $148M settlement |
| Hardcoded in source | Multiple startups on GitHub | Public credential exposure |
| Logs and debug output | Applications printing config | Credential in log aggregation |
| Container image layers | Docker images with secrets in ENV | Anyone with image access |
| Misconfigured K8s secrets | Unrestricted RBAC to secrets | Cluster-wide credential access |
| Stale/unused credentials | Former employee API keys still active | Unauthorized access |
| Shared secrets | Same DB password across 20 services | Lateral movement |

### Real-World Breaches

- **Uber (2016)**: AWS access keys hardcoded in a private GitHub repository were discovered by attackers. Cost: $148M settlement, CTO and CISO fired.
- **Tesla (2018)**: Kubernetes console misconfiguration led to cryptomining. AWS keys exposed via pod metadata.
- **Microsoft (2023)**: 38TB of internal data leaked due to misconfigured storage account with SAS token exposure.
- **RockYou2024**: 10 billion credentials leaked, many from hardcoded secrets in repositories.

## HashiCorp Vault

### Architecture

```mermaid
sequenceDiagram
    participant App as Banking Application
    participant VaultAgent as Vault Agent (Sidecar)
    participant Vault as Vault Server
    participant KMS as Cloud KMS / HSM
    participant DB as Database

    App->>VaultAgent: Read secret from mounted path
    VaultAgent->>Vault: Authenticate (Kubernetes auth)
    Vault->>Vault: Verify service account JWT
    Vault->>App: Return access token
    VaultAgent->>Vault: Request secret (with token)
    Vault->>Vault: Check policies
    Vault->>KMS: Decrypt secret (if encrypted storage)
    KMS->>Vault: Decrypted secret
    Vault->>VaultAgent: Secret with lease
    VaultAgent->>App: Write secret to shared memory
    App->>DB: Connect with dynamic credentials
```

### Vault Setup for Banking

```hcl
# Vault server configuration
storage "raft" {
  path    = "/vault/data"
  node_id = "vault-1"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_cert_file = "/vault/tls/server.crt"
  tls_key_file  = "/vault/tls/server.key"
  tls_min_version = "tls12"
}

api_addr     = "https://vault-1.vault-internal:8200"
cluster_addr = "https://vault-1.vault-internal:8201"

ui = true
```

### Kubernetes Authentication

```bash
# Enable Kubernetes auth method
vault auth enable kubernetes

# Configure Kubernetes auth
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token

# Create a role for the banking API service
vault write auth/kubernetes/role/banking-api \
  bound_service_account_names=banking-api-sa \
  bound_service_account_namespaces=production \
  policies=banking-api-policy \
  ttl=1h

# Define the policy
vault policy write banking-api-policy - <<EOF
path "secret/data/banking/db" {
  capabilities = ["read"]
}
path "secret/data/banking/api-keys" {
  capabilities = ["read"]
}
path "secret/data/banking/tls-certs" {
  capabilities = ["read"]
}
path "database/creds/banking-db" {
  capabilities = ["read"]
}
EOF
```

### Dynamic Database Credentials

```python
# BAD: Static database password in environment variable
import os
DB_PASSWORD = os.environ["DB_PASSWORD"]  # Leaked in .env file, container ENV, etc.

# GOOD: Vault dynamic credentials (short-lived, auto-revoked)
import hvac

class VaultDBCredentials:
    def __init__(self, vault_url: str, k8s_token_path: str):
        self.client = hvac.Client(url=vault_url)

        # Authenticate with Kubernetes service account
        with open(k8s_token_path) as f:
            k8s_token = f.read()

        self.client.auth.kubernetes.login(
            role="banking-api",
            jwt=k8s_token,
        )

    def get_db_credentials(self) -> dict:
        """
        Generate short-lived database credentials.
        Vault creates a unique username/password pair that auto-expires.
        """
        response = self.client.read("database/creds/banking-db")
        return {
            "username": response["data"]["username"],
            "password": response["data"]["password"],
            "ttl": response["lease_duration"],
        }

    def get_connection_string(self) -> str:
        creds = self.get_db_credentials()
        return f"postgresql://{creds['username']}:{creds['password']}@db.bank.internal:5432/banking"

# Usage in application
vault = VaultDBCredentials(
    vault_url="https://vault.vault.svc:8200",
    k8s_token_path="/var/run/secrets/kubernetes.io/serviceaccount/token"
)

# Get fresh credentials for each connection (or cache with TTL)
conn_string = vault.get_connection_string()
connection = psycopg2.connect(conn_string)
```

### Vault Database Configuration

```bash
# Configure PostgreSQL secrets engine
vault secrets enable database

# Configure the PostgreSQL connection
vault write database/config/banking-postgres \
  plugin_name=postgresql-database-plugin \
  allowed_roles="banking-db" \
  connection_url="postgresql://{{username}}:{{password}}@postgres.bank.svc:5432/banking?sslmode=require" \
  username="vault-admin" \
  password="VAULT_ADMIN_PASSWORD"

# Create a role that generates credentials
vault write database/roles/banking-db \
  db_name=banking-postgres \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"
```

### Vault Agent Sidecar

```yaml
# Inject Vault Agent as a sidecar container
apiVersion: apps/v1
kind: Deployment
metadata:
  name: banking-api
  namespace: production
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/agent-inject-secret-db-creds: "database/creds/banking-db"
        vault.hashicorp.com/agent-inject-template-db-creds: |
          {{- with secret "database/creds/banking-db" -}}
          export DB_USERNAME="{{ .Data.username }}"
          export DB_PASSWORD="{{ .Data.password }}"
          {{- end -}}
        vault.hashicorp.com/role: "banking-api"
        vault.hashicorp.com/tls-skip-verify: "false"
    spec:
      serviceAccountName: banking-api-sa
      containers:
      - name: banking-api
        image: registry.internal/banking-api:latest
        env:
        - name: VAULT_SECRET_PATH
          value: "/vault/secrets/db-creds"
        command: ["/bin/sh", "-c"]
        args:
        - |
          source /vault/secrets/db-creds
          exec /app/start-api
```

## Kubernetes Native Secrets

### When to Use K8s Secrets vs Vault

| Criteria | K8s Secrets | HashiCorp Vault |
|---|---|---|
| Secret type | Static credentials | Dynamic, short-lived credentials |
| Rotation | Manual | Automatic |
| Audit | Basic API audit | Detailed access logging |
| Encryption | etcd encryption (if enabled) | Auto-encryption with transit seal |
| Access control | K8s RBAC | Fine-grained policies |
| Use case | Bootstrap, low-security | Production banking workloads |

### Securing K8s Secrets

```yaml
# Enable encryption at rest for secrets
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources: ["secrets"]
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <BASE64_ENCODED_SECRET>
      - identity: {}  # Fallback (unencrypted)

# Restrict secret access with RBAC
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
  resourceNames: ["banking-api-secret"]  # Named secrets only

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: banking-api-secret-access
  namespace: production
subjects:
- kind: ServiceAccount
  name: banking-api-sa
  namespace: production
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io

# Secret definition (encrypted at rest by EncryptionConfiguration above)
apiVersion: v1
kind: Secret
metadata:
  name: banking-api-secret
  namespace: production
type: Opaque
data:
  api-key: c2tfbGl2ZV9hYmMxMjM=  # base64 encoded
stringData:
  config.yaml: |
    database:
      host: postgres.bank.svc
      port: 5432
```

### External Secrets Operator (ESO)

```yaml
# Sync secrets from Vault to K8s automatically
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault.vault.svc:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "external-secrets"
          serviceAccountRef:
            name: "external-secrets-sa"
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: banking-api-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: banking-api-secrets
    creationPolicy: Owner
    template:
      type: Opaque
  data:
  - secretKey: db-password
    remoteRef:
      key: secret/data/banking/db
      property: password
  - secretKey: api-key
    remoteRef:
      key: secret/data/banking/api-keys
      property: banking-api-key
```

## Secret Rotation

### Automated Rotation Pipeline

```python
# Vault auto-rotation for static secrets
import requests

def configure_rotation():
    """Configure automatic rotation for a static secret"""
    response = requests.post(
        f"{VAULT_ADDR}/v1/database/config/banking-postgres",
        headers={"X-Vault-Token": VAULT_TOKEN},
        json={
            "plugin_name": "postgresql-database-plugin",
            "allowed_roles": ["banking-db"],
            "connection_url": "postgresql://{{username}}:{{password}}@postgres.bank.svc:5432/banking",
            "username": "vault-admin",
            "password": "current_password",
            "root_rotation_statements": [
                "ALTER USER \"{{name}}\" WITH PASSWORD '{{password}}';"
            ],
            "rotate": True,  # Trigger immediate rotation
        }
    )
    response.raise_for_status()

# Set up rotation period (every 24 hours)
# In production, use Vault's built-in rotation or external automation
```

### Rotation Strategy by Secret Type

| Secret Type | Rotation Period | Method | Notification |
|---|---|---|---|
| Database passwords | 24h (dynamic) | Vault database secrets engine | None (auto) |
| API keys (internal) | 90 days | Vault KV + CI/CD pipeline | Slack alert 7 days before |
| TLS certificates | 90 days | cert-manager auto-renewal | Email 30 days before expiry |
| JWT signing keys | 180 days | Key rotation with overlap | Teams channel notification |
| Third-party API keys | Per provider policy | Manual update via PR | Calendar reminder |
| Service account tokens | 1h | K8s token auto-rotation | None (auto) |

## Access Control and Audit

### Vault Audit Logging

```bash
# Enable audit logging to file
vault audit enable file file_path=/vault/logs/audit.log

# Enable audit logging to syslog
vault audit enable syslog tag=vault

# Sample audit log entry
# {
#   "time": "2024-01-15T10:30:00.000000Z",
#   "type": "request",
#   "auth": {
#     "client_token": "hmac-sha256:abc123...",
#     "display_name": "kubernetes-production-banking-api",
#     "policies": ["banking-api-policy"],
#     "token_policies": ["banking-api-policy"],
#     "token_type": "service"
#   },
#   "request": {
#     "id": "req-123",
#     "operation": "read",
#     "path": "secret/data/banking/db",
#     "remote_address": "10.0.1.50"
#   },
#   "response": {
#     "data": {
#       "password": "hmac-sha256:xyz789..."  # Secret value is HMAC'd, never plaintext
#     }
#   }
# }
```

### Least Privilege Principle

```bash
# BAD: Broad policy allowing access to all secrets
vault policy write banking-bad-policy - <<EOF
path "secret/*" {
  capabilities = ["read", "list"]
}
EOF

# GOOD: Specific policy for specific secrets
vault policy write banking-api-policy - <<EOF
# Only read specific secret paths
path "secret/data/banking/db" {
  capabilities = ["read"]
}
path "secret/data/banking/api-keys" {
  capabilities = ["read"]
}

# Dynamic database credentials
path "database/creds/banking-db" {
  capabilities = ["read"]
}

# TLS certificates for service communication
path "pki/issue/banking-internal" {
  capabilities = ["create"]
  max_wrapped_ttl = "1h"
}

# NO list capability - don't reveal what other secrets exist
# NO create/update/delete - read only
EOF
```

## GenAI-Specific Secrets Management

### LLM API Key Security

```python
# BAD: Hardcoded OpenAI/Anthropic API key
import openai
openai.api_key = "sk-proj-abc123..."  # Committed to Git, leaked

# GOOD: Vault-backed API key
class LLMAPIKeyManager:
    def __init__(self, vault_client):
        self.vault = vault_client
        self._key_cache = {}
        self._cache_ttl = 300  # 5 minutes

    def get_llm_api_key(self, provider: str) -> str:
        """Get LLM API key from Vault with caching"""
        now = time.time()
        cached = self._key_cache.get(provider)

        if cached and now - cached["expires_at"] < self._cache_ttl:
            return cached["key"]

        # Fetch from Vault
        response = self.vault.read(f"secret/data/genai/{provider}")
        api_key = response["data"]["api_key"]

        # Cache with expiry
        self._key_cache[provider] = {
            "key": api_key,
            "expires_at": now + self._cache_ttl,
        }

        return api_key

    def rotate_api_key(self, provider: str, new_key: str):
        """Rotate LLM API key in Vault"""
        self.vault.write(
            f"secret/data/genai/{provider}",
            api_key=new_key,
            rotated_at=datetime.utcnow().isoformat(),
        )
        # Invalidate cache
        self._key_cache.pop(provider, None)

# Usage
key_manager = LLMAPIKeyManager(vault_client)
api_key = key_manager.get_llm_api_key("openai")

client = OpenAI(api_key=api_key)
response = client.chat.completions.create(...)
```

### Preventing LLM Secret Leakage

```python
# GenAI systems can accidentally leak secrets in prompts or outputs
import re

class SecretDetector:
    """Detect secrets in LLM prompts and responses"""

    PATTERNS = [
        (r'sk-[a-zA-Z0-9]{48}', 'OpenAI API key'),
        (r'sk-ant-[a-zA-Z0-9\-]+', 'Anthropic API key'),
        (r'AIza[a-zA-Z0-9\-_]{35}', 'Google API key'),
        (r'ghp_[a-zA-Z0-9]{36}', 'GitHub token'),
        (r'-----BEGIN (RSA )?PRIVATE KEY-----', 'Private key'),
        (r'AKIA[0-9A-Z]{16}', 'AWS access key'),
        (r'(?i)(password|passwd|pwd)\s*[:=]\s*\S+', 'Password in text'),
    ]

    @classmethod
    def scan_text(cls, text: str) -> list[dict]:
        """Scan text for potential secrets"""
        findings = []
        for pattern, secret_type in cls.PATTERNS:
            matches = re.findall(pattern, text)
            for match in matches:
                findings.append({
                    "type": secret_type,
                    "match": match[:8] + "..." + match[-4:],  # Don't log full secret
                    "severity": "CRITICAL",
                })
        return findings

    @classmethod
    def scan_llm_input(cls, prompt: str) -> bool:
        """Block prompts that might contain secrets"""
        findings = cls.scan_text(prompt)
        if findings:
            logger.critical("Potential secret in LLM prompt", findings=findings)
            return False  # Block the request
        return True

    @classmethod
    def scan_llm_output(cls, response: str) -> str:
        """Redact secrets from LLM output"""
        sanitized = response
        for pattern, secret_type in cls.PATTERNS:
            sanitized = re.sub(pattern, f"[REDACTED:{secret_type}]", sanitized)
        return sanitized
```

## Interview Questions

### Junior Level

1. Why should you never commit secrets to version control?
2. What is the difference between a secret and a configuration value?
3. What does it mean for a credential to be "dynamic"?
4. Why is storing secrets in environment variables problematic in Kubernetes?

### Senior Level

1. How does Vault's Kubernetes authentication work? Walk through the flow.
2. Design a secret rotation strategy for a system with 50 microservices and 200 secrets.
3. How do you prevent secrets from appearing in application logs?
4. What is the principle of least privilege in secrets access?

### Staff Level

1. How would you design a multi-region secrets management system with failover?
2. What is your strategy for responding to a confirmed secret leak in production?
3. How do you balance security (frequent rotation) with operational stability?

## Cross-References

- [Encryption](./encryption.md) - Encryption key management
- [Kubernetes Security](./kubernetes-security.md) - K8s secrets configuration
- [TLS and Certificates](./tls-and-certificates.md) - Certificate management
- [Secure Logging](./secure-logging.md) - Preventing secret leakage in logs
