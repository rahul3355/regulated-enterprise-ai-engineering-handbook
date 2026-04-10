# PCI-DSS (Payment Card Industry Data Security Standard)

## What Is PCI-DSS and Why It Matters

PCI-DSS is a security standard mandated by the Payment Card Industry Security Standards Council (founded by Visa, MasterCard, American Express, Discover, and JCB). It applies to **any organization** that stores, processes, or transmits cardholder data (CHD) or sensitive authentication data (SAD).

**Consequences of non-compliance**:
- Fines: $5,000 to $100,000 **per month** from card brands
- Increased transaction fees
- Loss of ability to process card payments
- Liability for all fraud and breach costs
- Mandatory forensic audits at your expense

For GenAI systems in banking: if your AI system processes, analyzes, or generates output involving card numbers (PAN), expiration dates, cardholder names, or any cardholder data, PCI-DSS applies.

### Key Definitions

| Term | Definition | Example |
|------|-----------|---------|
| **PAN** | Primary Account Number (the card number) | 4111 1111 1111 1111 |
| **CHD** | Cardholder Data -- PAN + cardholder name, expiration, service code | Full card details |
| **SAD** | Sensitive Authentication Data -- full track data, CAVV, CVC/CVV, PINs | CVV code |
| **CDE** | Cardholder Data Environment -- systems that store, process, or transmit CHD | Your AI service if it sees card numbers |
| **Scope** | All systems connected to the CDE | Networks, servers, databases, applications |

### Critical Rule: NEVER Store Sensitive Authentication Data

After authorization, you must **NEVER** store:
- Full magnetic stripe data (Track 1/Track 2)
- CAVV (Cardholder Authentication Verification Value)
- CVC2/CVV2/CID (the 3-4 digit security code)
- PINs or PIN blocks

**GenAI implication**: If your AI system receives card verification codes (e.g., a customer types their CVV into a chatbot), you must NOT log it, store it, or include it in any persistent data.

## PCI-DSS v4.0 -- The Current Version

PCI-DSS v4.0 became effective March 2024, with v3.2.1 retirement in March 2025. Key changes relevant to GenAI:

- **Requirement 6.3.2**: All payment page scripts must be protected (relevant for AI chat widgets)
- **Requirement 10.7.3**: Enhanced log review requirements
- **Requirement 11.4.3**: External vulnerability scanning requirements
- **New Appendix A3**: Multi-tenant service provider requirements
- **Customized approach**: Organizations can define their own security controls if they meet the intent

## The 12 PCI-DSS Requirements

### Requirement 1: Install and Maintain Network Security Controls

**What it means**: Firewalls and network segmentation to protect the CDE.

**Engineering requirements**:
- Isolate CDE from non-CDE networks using firewalls
- Document all data flows into and out of the CDE
- Restrict inbound/outbound traffic to only what is necessary
- Review firewall rules every 6 months

**GenAI architecture**:
```
GOOD:    AI service in isolated VPC, card data tokenized before entering AI pipeline
BAD:     AI service directly accesses production database with PANs
```

### Requirement 2: Apply Secure Configurations

**What it means**: No default passwords, no unnecessary services, hardened systems.

**Engineering requirements**:
- Document and implement secure configuration standards for all system components
- Change all vendor-supplied defaults (passwords, SNMP community strings, etc.)
- Disable all unnecessary services, protocols, and daemons
- Implement only one primary function per server (no multi-purpose CDE servers)

**Configuration baseline for AI services in CDE**:
```yaml
# Secure configuration example
ai_service:
  tls:
    min_version: "1.2"
    ciphers:
      - "TLS_AES_256_GCM_SHA384"
      - "TLS_CHACHA20_POLY1305_SHA256"
  authentication:
    require_mfa: true
    session_timeout: "15m"
  logging:
    mask_pan: true  # Only show last 4 digits
    exclude_cvv: true  # NEVER log CVV
  database:
    encryption_at_rest: true
    encryption_algorithm: "AES-256-GCM"
```

### Requirement 3: Protect Stored Account Data

**What it means**: If you store CHD, it must be protected.

**Engineering requirements**:

**Data you MAY store** (with protections):
- PAN (must be rendered unreadable)
- Cardholder name
- Expiration date
- Service code

**Data you MUST NOT store** (ever):
- Full track data (Track 1/Track 2)
- CAVV/CVC2/CVV2/CID
- PINs

**Rendering PAN unreadable** (use at least ONE of these):
1. One-way hashes based on strong cryptography (of the entire PAN)
2. Truncation (cannot store full PAN alongside hashed/truncated PAN)
3. Index tokens and pads (with the pads securely stored)
4. Strong cryptography (AES-256-RSA-2048+)

**Engineering implementation**:
```python
# PAN handling in AI system
import hashlib
import secrets
from cryptography.fernet import Fernet

class PANHandler:
    def __init__(self, encryption_key: bytes):
        self.fernet = Fernet(encryption_key)

    def tokenize_pan(self, pan: str) -> str:
        """Create a token for the PAN -- stored separately."""
        token = secrets.token_urlsafe(32)
        # Store mapping in secure token vault (HSM preferred)
        self.token_vault.store(token, self.encrypt_pan(pan))
        return token

    def encrypt_pan(self, pan: str) -> str:
        """Encrypt PAN for storage."""
        return self.fernet.encrypt(pan.encode()).decode()

    def mask_pan_for_display(self, pan: str) -> str:
        """Show only last 4 digits."""
        return f"****-****-****-{pan[-4:]}"

    def hash_pan(self, pan: str) -> str:
        """One-way hash for matching (cannot be reversed)."""
        # Use HMAC-SHA256 with a separate key for deterministic hashing
        import hmac
        return hmac.new(self.hash_key, pan.encode(), hashlib.sha256).hexdigest()

    # NEVER: store CVV
    def validate_cvv_not_stored(self, system_audit):
        """Verify no CVV data exists in any storage."""
        for storage in self.all_storage_systems():
            assert not storage.contains("cvv", "cvc", "card_verification")
```

**Key management requirements**:
- Encryption keys must be stored separately from encrypted data
- Key-encrypting keys must be at least as strong as data-encrypting keys
- Keys must be changed on schedule and when compromised
- Key management procedures must be documented
- Split knowledge and dual control for key management

### Requirement 4: Protect Cardholder Data During Transmission

**What it means**: Encrypt CHD during transmission over open, public networks.

**Engineering requirements**:
- Use TLS 1.2 or higher for all transmissions
- Verify certificate validity (no self-signed certs in production)
- Only accept trusted certificates with proper chain of trust
- Ensure encryption is used for all management interfaces
- Do NOT send PANs in unencrypted emails, chats, or instant messages

**GenAI-specific concerns**:
```
PROHIBITED:
  - Sending PAN in prompt to external AI API without encryption
  - Logging full PAN in application logs
  - Including PAN in URL parameters
  - Sending PAN via internal messaging (Slack, Teams, etc.)
  - Storing PAN in browser localStorage

REQUIRED:
  - Tokenize PAN before sending to AI service
  - Use mTLS for service-to-service communication in CDE
  - Encrypt PAN in all database fields
  - Mask PAN in UI (show only last 4 digits)
```

### Requirement 5: Protect Against Malware

**What it means**: Deploy anti-malware on all systems (especially those outside the CDE that could access it).

**Engineering requirements**:
- Deploy endpoint detection and response (EDR) on all servers
- Keep anti-malware signatures updated
- Perform periodic scans
- Protect against phishing attacks

### Requirement 6: Develop and Maintain Secure Systems and Software

**What it means**: Secure SDLC, vulnerability management, change management.

**Engineering requirements**:

**Secure SDLC**:
- Train developers in secure coding annually
- Code reviews must include security review
- WAF or equivalent for all internet-facing web applications
- Address OWASP Top 10 vulnerabilities

**Vulnerability management**:
- Identify new security vulnerabilities quarterly
- Rank vulnerabilities using CVSS scoring
- Apply security patches within specific timeframes:
  - Critical: within 1 month
  - High: within 3 months
- Internal and external vulnerability scans quarterly
- Penetration testing annually and after significant changes

**GenAI-specific security considerations**:
```
AI-SPECIFIC VULNERABILITY CLASSES:
  - Prompt injection (OWASP LLM Top 10 #1)
  - Training data poisoning (OWASP LLM Top 10 #3)
  - Insecure output handling (OWASP LLM Top 10 #4)
  - Excessive agency (OWASP LLM Top 10 #5)
  - Overreliance on AI output (OWASP LLM Top 10 #9)

MITIGATIONS:
  - Input validation and sanitization on all prompts
  - Output validation before displaying to users
  - Content filtering on AI responses
  - Human review for AI-generated decisions
  - Rate limiting on AI endpoints
```

### Requirement 7: Restrict Access by Business Need to Know

**What it means**: Only authorized personnel can access CHD.

**Engineering requirements**:
- Define access roles based on job function
- Document required access for each role
- Implement least privilege
- Annual access review

```python
# RBAC for CDE
class AccessControl:
    ROLES = {
        "customer_service": {
            "can_view_masked_pan": True,
            "can_view_full_pan": False,
            "can_access_ai_chat": True,
            "can_access_transaction_history": True,
        },
        "fraud_analyst": {
            "can_view_masked_pan": True,
            "can_view_full_pan": True,  # With audit trail
            "can_access_ai_chat": True,
            "can_access_transaction_history": True,
        },
        "ai_engineer": {
            "can_view_masked_pan": False,
            "can_view_full_pan": False,  # NEVER
            "can_access_ai_chat": True,  # Anonymized only
            "can_access_transaction_history": False,
        },
        "auditor": {
            "can_view_masked_pan": True,
            "can_view_full_pan": False,
            "can_access_ai_chat": True,  # Redacted
            "can_access_transaction_history": True,
            "can_view_audit_logs": True,
        },
    }
```

### Requirement 8: Identify Users and Authenticate Access

**What it means**: Unique IDs, strong authentication, secure password policies.

**Engineering requirements**:
- Unique user ID for every person with access
- Multi-factor authentication for all CDE access
- Password complexity requirements (min 12 characters, complexity)
- Account lockout after failed attempts
- Session timeout after 15 minutes of inactivity
- Do NOT use group/shared accounts for CDE access

### Requirement 9: Restrict Physical Access

**What it means**: Physical access controls for systems in the CDE.

**Engineering requirements** (for on-prem infrastructure):
- Badge access to data centers
- Visitor logs and escorts
- CCTV monitoring
- Secure media disposal

### Requirement 10: Log and Monitor All Access

**What it means**: Comprehensive audit logging for the CDE.

**Engineering requirements**:

**Must log**:
- All individual user accesses to CHD
- All actions taken by any individual with root/administrative privileges
- Access to all audit trails
- Invalid logical access attempts
- Changes to identification and authentication controls
- Changes to audit trail configurations
- Initialization, pausing, or stopping of audit logs
- Creation and deletion of system-level objects

**Log protection**:
- Logs must be write-once or append-only
- Logs must be backed up to centralized log server
- Logs must be protected from modification
- Access to logs restricted to authorized personnel

**Log review**:
- Review logs daily (or use automated log monitoring)
- Review exceptions/alerts
- Follow up on anomalies

```python
# Audit logging for CDE
class CDEAuditLogger:
    def log_access(self, user_id: str, action: str, resource: str, pan: str = None):
        """Log every access to CHD."""
        log_entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "user_id": user_id,
            "action": action,
            "resource": resource,
            "pan_reference": self.hash_pan_ref(pan) if pan else None,  # Never log PAN
            "source_ip": get_client_ip(),
            "session_id": get_session_id(),
            "result": "success" or "failure",
        }
        # Write to append-only audit log
        self.audit_log.append(log_entry)
        # Also ship to centralized SIEM
        self.siem_client.send(log_entry)

    def log_admin_action(self, admin_id: str, action: str, details: str):
        """Log all admin actions with enhanced detail."""
        log_entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "admin_id": admin_id,
            "action": action,
            "details": details,
            "privilege_level": "root" or "admin",
            "mfa_verified": verify_mfa_status(admin_id),
        }
        self.audit_log.append(log_entry)
```

**Retention**: Audit logs must be retained for at least 1 year, with 3 months immediately available for analysis.

### Requirement 11: Test Security Systems and Processes

**What it means**: Regular testing of security controls.

**Engineering requirements**:
- Vulnerability scans quarterly (internal and external)
- Penetration testing annually and after significant changes
- Intrusion detection/prevention systems (IDS/IPS)
- File integrity monitoring (FIM) on critical system files
- Change detection mechanisms for payment pages

**Testing methods**:
- Network-layer penetration tests
- Application-layer penetration tests
- Configuration reviews
- Network architecture reviews

### Requirement 12: Support Information Security with Policies

**What it means**: Documented security policies, risk assessment, incident response.

**Engineering requirements**:
- Annual risk assessment
- Information security policy documented and published
- Acceptable use policies for technology assets
- Incident response plan tested and ready
- Security awareness training for all personnel
- Background checks for new hires
- Vendor risk management program

## CDE Scoping for GenAI Systems

### Determining If Your AI System Is In Scope

Your GenAI system is **in scope** for PCI-DSS if it:

1. **Stores** cardholder data (even temporarily)
2. **Processes** cardholder data (analyzes, transforms, generates output from)
3. **Transmits** cardholder data (sends to another system, including AI APIs)
4. **Is connected to** any system that stores, processes, or transmits cardholder data

### Scoping Strategies

**Strategy 1: Reduce Scope via Tokenization**

```
Customer provides PAN -> Tokenize immediately -> Use tokens in AI pipeline
                                                    │
                                          AI sees only tokens
                                          (no PAN exposure)
                                          CDE scope minimized
```

**Strategy 2: Network Segmentation**

```
Segment A: CDE (payment processing, card storage)
    │
    └── Firewall (strictly controlled)
        │
        Segment B: AI Service (receives tokenized data)
            │
            └── Firewall
                │
                Segment C: Public-facing chatbot
```

**Strategy 3: Outsource Card Handling**

```
Customer -> Payment iframe (hosted by payment processor) -> Token returned to your system
                                                                     │
                                                           Your AI system processes
                                                           tokens, never PANs
                                                           Reduced PCI scope
```

### Out-of-Scope AI Systems

If your AI system never handles cardholder data, it may be out of scope -- but you must prove it:

1. Document that CHD is never sent to the AI system
2. Implement input validation to reject CHD
3. Monitor for accidental CHD submission
4. Network segmentation documented and tested
5. Annual validation of scope

## Common Violations and Real-World Incidents

### Notable PCI-DSS Breaches

| Organization | Year | Impact | Details |
|-------------|------|--------|---------|
| Capital One | 2019 | $190M fine | Misconfigured WAF, 100M+ card applications exposed |
| British Airways | 2018 | £20M fine | Card skimming via Magecart, 400K+ cards compromised |
| Marriott | 2020 | £18.4M fine | Payment card data exposed in breach |
| Uber | 2018 | $148M settlement | Covered up breach affecting 57M users |
| Equifax | 2017 | $575M+ settlement | Unpatched vulnerability, 147M records including payment data |
| Target | 2013 | $18.5M settlement | POS malware, 40M cards compromised |

### Common Violation Categories

1. **Storing CVV/CVC codes** -- Even temporarily is prohibited
2. **Logging full PANs** -- In application logs, debug output, or error messages
3. **Using default passwords** -- On CDE systems
4. **Not segmenting networks** -- CDE connected to non-CDE without firewall
5. **Not encrypting stored PANs** -- PANs stored in plaintext or weakly encrypted
6. **Not reviewing logs** -- Logs exist but no one reviews them
7. **Shared accounts** -- Multiple people using the same admin account
8. **Outdated TLS** -- Using TLS 1.0/1.1 or weak ciphers

## Required Controls Checklist

### For Any System That May Encounter Cardholder Data

- [ ] PAN detection and auto-redaction in all inputs
- [ ] CVV rejection and non-storage enforcement
- [ ] Encryption of any stored PAN (AES-256 minimum)
- [ ] Key management with HSM or equivalent
- [ ] Unique user IDs for all access
- [ ] MFA for all CDE access
- [ ] Comprehensive audit logging (append-only)
- [ ] Log review and monitoring
- [ ] Annual penetration testing
- [ ] Quarterly vulnerability scanning
- [ ] Network segmentation documentation
- [ ] Incident response plan tested
- [ ] Vendor risk assessments completed
- [ ] Secure coding training completed

### For GenAI Systems Specifically

- [ ] Prompt input validation rejects full PANs
- [ ] CVV is never included in prompts or stored
- [ ] AI outputs do not reproduce cardholder data
- [ ] Tokenization used instead of raw PANs where possible
- [ ] External AI API calls use encrypted channels
- [ ] AI training data does not contain unmasked PANs
- [ ] Vector database embeddings do not encode PANs
- [ ] Prompt logs are scanned for accidental PAN exposure
- [ ] AI model cannot be prompted to reveal stored PANs

## How PCI-DSS Affects Each System Component

### APIs
- Must authenticate and authorize every request
- Must mask PAN in all API responses (show only last 4 digits)
- Must never accept CVV in API payloads (or must immediately discard)
- Must log all access with unique user IDs
- Must use TLS 1.2+ for all connections

### Frontend
- Must not store PAN in localStorage, sessionStorage, or cookies
- Must mask PAN in all displays (****-****-****-1234)
- Must use secure, PCI-compliant payment forms (hosted fields or iframes)
- Must implement secure session management
- Must not expose card data in browser developer tools

### Backend
- Must encrypt PAN at rest
- Must never log PAN or CVV
- Must implement tokenization
- Must enforce access controls on all card data access
- Must have key management procedures

### Databases
- Must encrypt all PAN columns at rest
- Must implement column-level access controls
- Must log all access to PAN-containing tables
- Must support PAN masking in query results
- Must never store CVV

### CI/CD
- Must scan for hardcoded card numbers in code
- Must require security review for CDE changes
- Must include penetration testing in release process
- Must validate infrastructure configurations

### Monitoring
- Must detect unauthorized access to CHD
- Must alert on anomalous access patterns
- Must monitor for PAN in logs
- Must track authentication failures

### AI Systems
- Must validate AI inputs for PAN/CVV
- Must not reproduce PAN in AI outputs
- Must use tokenized data for training
- Must document AI model's access to cardholder data
- Must validate AI cannot be prompted to reveal PANs

### Prompt Logging
- Must scan and redact PANs from logged prompts
- Must never log CVV
- Must mask any accidentally captured PANs
- Must apply same retention as other audit data

### Vector Databases
- Must not embed raw PANs into vectors
- Must use tokenized or hashed identifiers
- Must implement access controls
- Must encrypt at rest

## Common Interview Questions

### Question 1: "How do you build a PCI-compliant chatbot for payment inquiries?"

**What they're testing**: Understanding of scope reduction, data handling, and logging.

**Good answer structure**:
1. Tokenize PANs immediately -- chatbot never sees real card numbers
2. Never ask for or store CVV in chat
3. Mask PAN in all displays (last 4 digits only)
4. Do not log card numbers in conversation logs
5. Authenticate users before discussing account details
6. Log all chatbot access for audit purposes
7. Implement input validation to reject accidental card number input
8. Use network segmentation to isolate chatbot from CDE where possible

### Question 2: "Can you store transaction data that includes masked card numbers in your vector database for RAG?"

**What they're testing**: Understanding of PCI scope for AI systems.

**Good answer structure**:
1. Masked PANs (last 4 digits only) are generally considered out of PCI scope
2. However, if masked PAN + other data can reconstruct full PAN, it's still in scope
3. Best practice: Use tokenized references, not even masked PANs, in embeddings
4. Implement access controls on the vector database
5. Document the data flow and get QSA approval
6. Regular validation that no full PANs exist in the vector store

### Question 3: "What's the difference between being PCI-DSS compliant and being PCI-DSS certified?"

**What they're testing**: Practical knowledge of compliance vs. certification.

**Good answer structure**:
1. PCI-DSS doesn't have a formal "certification" -- it's about being compliant
2. Organizations complete a Self-Assessment Questionnaire (SAQ) or Report on Compliance (ROC)
3. Level 1 merchants (>6M transactions/year) require annual ROC by QSA
4. Lower levels can complete SAQ
5. Compliance is continuous, not a one-time event
6. Acquiring banks and card brands enforce compliance
