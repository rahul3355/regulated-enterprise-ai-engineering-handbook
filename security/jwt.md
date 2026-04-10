# JWT Security

## Overview

JSON Web Tokens (JWT) are the standard for securely transmitting claims between parties in banking systems. A single misconfiguration -- wrong algorithm, missing validation, or exposed signing key -- can lead to complete authentication bypass. This guide covers JWT internals, common vulnerabilities, and production-hardened handling patterns.

## JWT Structure

A JWT consists of three Base64URL-encoded parts separated by dots:

```
header.payload.signature
```

### Header

```json
{
  "alg": "RS256",     // Signing algorithm
  "typ": "JWT",        // Token type
  "kid": "key-2024-01" // Key ID (for key rotation)
}
```

### Payload (Claims)

```json
{
  "iss": "https://auth.bank.com",           // Issuer
  "sub": "user-12345",                       // Subject (user ID)
  "aud": "banking-api",                      // Audience
  "exp": 1700003600,                         // Expiration time
  "iat": 1700000000,                         // Issued at
  "jti": "unique-token-id",                  // JWT ID (for revocation)
  "scope": "accounts:read transactions:read", // OAuth2 scopes
  "roles": ["customer"],                     // Custom claims
  "mfa_verified": true                       // Custom claims
}
```

### Signature

The signature is computed by:
1. Taking the Base64URL-encoded header and payload
2. Joining them with a period: `encodedHeader.encodedPayload`
3. Signing with the private key (RS256) or shared secret (HS256)

## Algorithm Comparison

| Algorithm | Type | Key Size | Performance | Banking Recommendation |
|---|---|---|---|---|
| HS256 | Symmetric | 256-bit | Fast | Avoid (shared secret = single point of compromise) |
| HS384 | Symmetric | 384-bit | Fast | Avoid |
| HS512 | Symmetric | 512-bit | Fast | Avoid |
| RS256 | Asymmetric | 2048-bit+ RSA | Moderate | Recommended (widely supported) |
| RS384 | Asymmetric | 2048-bit+ RSA | Moderate | Acceptable |
| RS512 | Asymmetric | 2048-bit+ RSA | Moderate | Acceptable |
| ES256 | Asymmetric | 256-bit EC | Fast | Recommended (smaller keys, faster) |
| ES384 | Asymmetric | 384-bit EC | Moderate | Good for high security |
| ES512 | Asymmetric | 512-bit EC | Moderate | Overkill for most cases |
| PS256 | Asymmetric | 2048-bit+ RSA | Moderate | Good (RSA-PSS, more secure than RS256) |
| none | None | N/A | N/A | NEVER ALLOW (bypasses signature) |

## Common JWT Vulnerabilities

### 1. Algorithm Confusion Attack (CVE-2015-9235)

**Attack**: An attacker changes the algorithm from RS256 (asymmetric) to HS256 (symmetric) and signs the token with the public key. The server, confused about which key to use, verifies the signature with the public key (which the attacker has), accepting a forged token.

```python
# ATTACK SCENARIO
# Server uses RS256 with public/private key pair
# Attacker has access to the public key (it's public!)

import jwt
import base64

# Attacker forges a token:
# 1. Takes a legitimate token and modifies claims
fake_payload = {
    "sub": "admin",  # Changed from "user-12345"
    "roles": ["admin"],
    "exp": 9999999999
}

# 2. Changes algorithm to HS256
# 3. Signs with the PUBLIC KEY (treating it as HMAC secret)
with open("public.pem", "rb") as f:
    public_key = f.read()

forged_token = jwt.encode(
    fake_payload,
    public_key,  # Using public key as HMAC secret!
    algorithm="HS256"
)

# 4. Server sees alg: HS256, uses "secret" (public key) to verify
# Since the attacker signed with the public key, verification passes!
```

**Prevention**:

```python
# ALWAYS explicitly specify allowed algorithms
import jwt

def validate_jwt(token: str, public_key: str) -> dict:
    """
    Securely validate a JWT token.
    """
    try:
        payload = jwt.decode(
            token,
            public_key,
            algorithms=["RS256"],  # EXPLICIT: Never accept "none" or dynamic algorithms
            audience="banking-api",
            issuer="https://auth.bank.com",
            options={
                "verify_exp": True,
                "verify_iat": True,
                "verify_aud": True,
                "verify_iss": True,
                "verify_signature": True,
                "require": ["exp", "aud", "iss", "sub"],  # Require critical claims
            }
        )
        return payload
    except jwt.ExpiredSignatureError:
        raise AuthenticationError("Token expired")
    except jwt.InvalidAudienceError:
        raise AuthenticationError("Invalid audience")
    except jwt.InvalidIssuerError:
        raise AuthenticationError("Invalid issuer")
    except jwt.InvalidTokenError as e:
        raise AuthenticationError(f"Invalid token: {e}")

# Additional: Reject "none" algorithm at the gateway level
# before the token even reaches the application
def reject_none_algorithm(token: str):
    header = json.loads(base64.urlsafe_b64decode(
        token.split('.')[0] + '=='
    ))
    if header.get('alg', '').lower() == 'none':
        raise AuthenticationError("Algorithm 'none' is not allowed")
```

### 2. Missing Expiration Validation

```python
# BAD: Not verifying expiration
payload = jwt.decode(token, secret, algorithms=["HS256"], options={"verify_exp": False})

# BAD: No expiration in options
payload = jwt.decode(token, public_key, algorithms=["RS256"])

# GOOD: Require and verify expiration
payload = jwt.decode(
    token,
    public_key,
    algorithms=["RS256"],
    options={"require": ["exp"]}
)
```

### 3. Weak Signing Key

```python
# BAD: Short, predictable secret
SECRET = "mysecret123"  # Crackable in seconds

# GOOD: Use a strong, randomly generated key
import secrets
SECRET = secrets.token_urlsafe(64)  # 64 bytes of randomness

# BETTER: Use asymmetric keys (RS256/ES256)
from cryptography.hazmat.primitives.asymmetric import rsa, padding
from cryptography.hazmat.primitives import hashes, serialization

def generate_rsa_keypair():
    private_key = rsa.generate_private_key(
        public_exponent=65537,
        key_size=2048,
    )
    public_key = private_key.public_key()

    private_pem = private_key.private_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PrivateFormat.PKCS8,
        encryption_algorithm=serialization.NoEncryption()
    )
    public_pem = public_key.public_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PublicFormat.SubjectPublicKeyInfo
    )

    return private_pem, public_pem
```

### 4. Storing Sensitive Data in Payload

```python
# BAD: Sensitive data in JWT payload (base64 is NOT encryption)
token = jwt.encode({
    "sub": "user-123",
    "ssn": "123-45-6789",          # NEVER put PII in JWT
    "credit_card": "4111111111111111",  # NEVER
    "password_hash": "$2b$12$...",     # NEVER
}, private_key, algorithm="RS256")

# The payload is easily decoded:
import base64
payload_part = token.split('.')[1]
decoded = base64.urlsafe_b64decode(payload_part + '==')
print(json.loads(decoded))  # All sensitive data exposed!

# GOOD: Only store non-sensitive identifiers
token = jwt.encode({
    "sub": "user-123",
    "roles": ["customer"],
    "mfa_verified": True,
    "exp": expiry_timestamp,
}, private_key, algorithm="RS256")

# Look up sensitive data from database using sub claim
def get_user_data(token: str):
    payload = validate_jwt(token, public_key)
    user_id = payload["sub"]
    return db.get_user(user_id)  # Sensitive data stays in database
```

### 5. Token Replay

```python
# PROBLEM: JWT has no built-in revocation mechanism
# If a token is stolen, it's valid until expiration

# SOLUTION: Use jti (JWT ID) claim with a revocation list
import jwt
import secrets
import time
from redis import Redis

class JWTManager:
    def __init__(self, redis: Redis, private_key: bytes, public_key: bytes):
        self.redis = redis
        self.private_key = private_key
        self.public_key = public_key
        self.token_ttl = 3600  # 1 hour

    def create_token(self, user_id: str, scopes: list) -> str:
        jti = secrets.token_urlsafe(16)  # Unique token ID

        payload = {
            "iss": "https://auth.bank.com",
            "sub": user_id,
            "aud": "banking-api",
            "exp": int(time.time()) + self.token_ttl,
            "iat": int(time.time()),
            "jti": jti,  # JWT ID for revocation
            "scope": " ".join(scopes),
        }

        return jwt.encode(payload, self.private_key, algorithm="RS256")

    def revoke_token(self, jti: str):
        """Add token to revocation list until it expires"""
        self.redis.setex(f"revoked:{jti}", self.token_ttl, "1")

    def validate_token(self, token: str) -> dict:
        payload = jwt.decode(
            token,
            self.public_key,
            algorithms=["RS256"],
            options={"require": ["exp", "aud", "iss", "sub", "jti"]},
        )

        # Check revocation list
        jti = payload["jti"]
        if self.redis.get(f"revoked:{jti}"):
            raise AuthenticationError("Token has been revoked")

        return payload

    def revoke_all_user_tokens(self, user_id: str):
        """
        Called on password change or suspected compromise.
        Requires maintaining a mapping of user_id -> active jtis.
        """
        # Option 1: Store all active jtis per user in Redis set
        # Option 2: Version the user's "token version" and check against it
        # Option 3: Use a shorter token TTL + refresh token rotation
        pass
```

## JWE (Encrypted JWT)

When claims must be encrypted (not just signed):

```python
from jwcrypto import jwt, jwk

# Create encrypted JWT
def create_encrypted_token(user_id: str, public_key: jwk.JWK) -> str:
    header = {"alg": "RSA-OAEP-256", "enc": "A256GCM"}
    payload = {
        "sub": user_id,
        "roles": ["customer"],
        "exp": int(time.time()) + 3600,
    }

    token = jwt.JWT(
        header=header,
        claims=json.dumps(payload)
    )
    token.make_encrypted_token(public_key)
    return token.serialize()

# Decrypt and validate
def decrypt_and_validate_token(token_str: str, private_key: jwk.JWK) -> dict:
    token = jwt.JWT(key=private_key, jwt=token_str)
    payload = json.loads(token.claims)
    return payload
```

## JWT in Kubernetes/OpenShift

### JWKS Endpoint for Key Distribution

```yaml
# Service exposing the JSON Web Key Set
apiVersion: v1
kind: Service
metadata:
  name: auth-jwks
  namespace: identity
spec:
  selector:
    app: auth-server
  ports:
    - port: 443
      targetPort: 8443
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auth-jwks
  namespace: identity
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - auth.bank.com
      secretName: auth-tls
  rules:
    - host: auth.bank.com
      http:
        paths:
          - path: /.well-known/jwks.json
            pathType: Prefix
            backend:
              service:
                name: auth-jwks
                port:
                  number: 443
```

### JWKS Response Format

```json
{
  "keys": [
    {
      "kty": "RSA",
      "use": "sig",
      "kid": "key-2024-01",
      "alg": "RS256",
      "n": "wqS8...base64url-encoded-modulus",
      "e": "AQAB"
    },
    {
      "kty": "RSA",
      "use": "sig",
      "kid": "key-2024-02",
      "alg": "RS256",
      "n": "xRT9...new-key-modulus",
      "e": "AQAB"
    }
  ]
}
```

### Key Rotation Strategy

```python
class KeyRotator:
    """
    Rotate signing keys without disrupting active tokens.

    Strategy:
    1. Generate new key pair
    2. Publish new public key to JWKS endpoint
    3. Sign new tokens with new key (new kid)
    4. Old tokens (with old kid) still validate until expiry
    5. After all old tokens expire, remove old public key from JWKS
    6. Delete old private key
    """

    def rotate_key(self):
        # 1. Generate new key
        new_private_key, new_public_key = generate_rsa_keypair()
        new_kid = f"key-{int(time.time())}"

        # 2. Store new private key securely (Vault)
        vault.write(f"secret/jwt-signing-keys/{new_kid}", private_key=new_private_key)

        # 3. Update JWKS endpoint with new public key
        jwks = self.load_jwks()
        jwks["keys"].append({
            "kty": "RSA",
            "use": "sig",
            "kid": new_kid,
            "alg": "RS256",
            "n": base64url_encode(new_public_key.modulus),
            "e": base64url_encode(new_public_key.exponent),
        })
        self.publish_jwks(jwks)

        # 4. Start using new key for signing
        self.current_signing_key = new_private_key
        self.current_kid = new_kid

        # 5. Schedule old key removal (after max token TTL)
        schedule_removal(old_kid, delay=3600)  # 1 hour
```

## Banking-Specific JWT Controls

### Custom Claims for Banking

```python
def create_banking_token(user: User, session: Session) -> str:
    return jwt.encode({
        "iss": "https://auth.acmebank.com",
        "sub": str(user.id),
        "aud": "acme-banking-api",
        "exp": int(time.time()) + 900,  # 15 minutes (short-lived for banking)
        "iat": int(time.time()),
        "jti": session.id,
        "scope": "accounts:read transactions:read",

        # Banking-specific claims
        "user_tier": user.tier,           # retail, premium, business
        "mfa_level": session.mfa_level,   # none, sms, app, hardware
        "session_fingerprint": session.device_fingerprint,
        "geo_location": session.geo_hash,  # For impossible travel detection
        "txn_limit": user.daily_txn_limit,
        "requires_step_up": True,          # Flag for sensitive operations
    }, private_key, algorithm="RS256", headers={"kid": current_kid})
```

### JWT Validation Middleware

```python
from functools import wraps
from fastapi import Request, HTTPException

class JWTAuthMiddleware:
    def __init__(self, jwks_client):
        self.jwks_client = jwks_client
        self.revocation_store = Redis()

    async def __call__(self, request: Request, call_next):
        auth_header = request.headers.get("Authorization")
        if not auth_header or not auth_header.startswith("Bearer "):
            raise HTTPException(401, "Missing or invalid Authorization header")

        token = auth_header.split(" ", 1)[1]

        try:
            # Get kid from token header to select correct key
            unverified_header = json.loads(
                base64.urlsafe_b64decode(token.split('.')[0] + '==')
            )
            kid = unverified_header.get("kid")
            if not kid:
                raise HTTPException(401, "Missing key ID in token")

            # Get signing key from JWKS
            signing_key = self.jwks_client.get_signing_key_from_jwt(token)

            # Decode and validate
            payload = jwt.decode(
                token,
                signing_key.key_info,
                algorithms=["RS256"],
                audience="acme-banking-api",
                issuer="https://auth.acmebank.com",
                options={"require": ["exp", "aud", "iss", "sub", "jti"]},
            )

            # Check revocation
            if self.revocation_store.get(f"revoked:{payload['jti']}"):
                raise HTTPException(401, "Token revoked")

            # Check device fingerprint match (session hijacking detection)
            current_fingerprint = generate_device_fingerprint(request)
            if payload.get("session_fingerprint") != current_fingerprint:
                # Log suspicious activity
                logger.warning(
                    "Possible session hijacking",
                    extra={"user_id": payload["sub"], "jti": payload["jti"]}
                )
                # In strict mode, reject. In monitoring mode, just log.
                raise HTTPException(401, "Session mismatch")

            request.state.user = payload
            return await call_next(request)

        except jwt.ExpiredSignatureError:
            raise HTTPException(401, "Token expired")
        except jwt.InvalidTokenError as e:
            raise HTTPException(401, f"Invalid token: {e}")
```

## Real-World JWT Vulnerabilities

- **CVE-2023-30581**: Critical vulnerability in node-jsonwebtoken library allowed signature bypass via algorithm confusion.
- **Auth0 (2023)**: Vulnerability in JWT validation allowed authentication bypass for some tenants.
- **Multiple Kubernetes API servers**: Default JWT configurations with missing audience validation allowed cross-cluster authentication.

## Security Testing

```python
# JWT security test cases
import pytest

class TestJWTSecurity:
    def test_none_algorithm_rejected(self):
        """Ensure 'none' algorithm is never accepted"""
        payload = {"sub": "admin", "exp": 9999999999}
        token = jwt.encode(payload, "", algorithm=None)
        with pytest.raises(jwt.InvalidTokenError):
            jwt.decode(token, public_key, algorithms=["RS256"])

    def test_algorithm_confusion_prevented(self):
        """HS256 token signed with public key should be rejected"""
        token = jwt.encode({"sub": "admin"}, public_key_pem, algorithm="HS256")
        with pytest.raises(jwt.InvalidTokenError):
            # Server only accepts RS256
            jwt.decode(token, public_key_pem, algorithms=["RS256"])

    def test_expired_token_rejected(self):
        token = jwt.encode(
            {"sub": "user", "exp": int(time.time()) - 100},
            private_key,
            algorithm="RS256"
        )
        with pytest.raises(jwt.ExpiredSignatureError):
            jwt.decode(token, public_key, algorithms=["RS256"])

    def test_invalid_audience_rejected(self):
        token = jwt.encode(
            {"sub": "user", "aud": "other-api", "exp": int(time.time()) + 3600},
            private_key,
            algorithm="RS256"
        )
        with pytest.raises(jwt.InvalidAudienceError):
            jwt.decode(token, public_key, algorithms=["RS256"], audience="banking-api")

    def test_revoked_token_rejected(self):
        token = jwt.encode(
            {"sub": "user", "jti": "revoked-123", "exp": int(time.time()) + 3600},
            private_key,
            algorithm="RS256"
        )
        redis.set("revoked:revoked-123", "1")
        # Custom check in validation middleware
        assert is_token_revoked("revoked-123")
```

## Interview Questions

### Junior Level

1. What are the three parts of a JWT and what does each contain?
2. Can you read the payload of a JWT? Is it encrypted?
3. What does the `alg` field in the header mean?
4. Why is it dangerous to store sensitive data in a JWT payload?

### Senior Level

1. Explain the algorithm confusion attack and how to prevent it.
2. How do you implement JWT revocation when JWTs are stateless by design?
3. What is the purpose of the `kid` (Key ID) field? How does it help with key rotation?
4. Why should you always explicitly specify allowed algorithms when decoding JWTs?

### Staff Level

1. Design a JWT key rotation strategy for a system with millions of users where tokens have a 1-hour TTL.
2. How would you detect and respond to a JWT signing key compromise?
3. When would you choose JWE (encrypted JWT) over JWS (signed JWT)? What are the trade-offs?

## Cross-References

- [OAuth2 and OIDC](./oauth2-and-oidc.md) - Protocols that use JWT
- [Authentication and Authorization](./authn-and-authz.md) - Token lifecycle
- [API Security](./api-security.md) - API token validation
- [Secrets Management](./secrets-management.md) - Protecting signing keys
