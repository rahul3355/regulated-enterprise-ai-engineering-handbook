# PII Handling

## What Is PII and Why It Matters

Personally Identifiable Information (PII) is any data that can identify a specific individual. In banking, virtually all customer data is PII. Mishandling PII can result in regulatory fines, data breach costs, and reputational damage.

**PII in the context of GenAI systems**:
- Customer prompts to AI chatbots may contain PII
- AI responses may contain or generate PII
- Vector embeddings may encode PII
- Training data for AI models contains PII
- Fine-tuning datasets contain PII
- AI system logs may capture PII

## Types of PII

### Direct Identifiers

Data that uniquely identifies an individual on its own:

| Data Type | Examples | Sensitivity |
|-----------|---------|-------------|
| Government IDs | SSN, passport number, driver's license, national insurance number | CRITICAL |
| Financial IDs | Bank account number, credit card number, tax ID | CRITICAL |
| Biometric data | Fingerprints, facial recognition, voice prints | CRITICAL |
| Contact information | Email, phone number, home address | HIGH |
| Full name | First + last name (when combined with other data) | HIGH |
| Date of birth | Full date of birth | HIGH |

### Indirect Identifiers

Data that can identify an individual when combined with other data:

| Data Type | Examples | Sensitivity |
|-----------|---------|-------------|
| Partial identifiers | Last 4 digits of SSN, ZIP code | MEDIUM |
| Demographics | Age, gender, ethnicity (when combined) | MEDIUM |
| Transaction data | Purchase history, transaction amounts | HIGH |
| Behavioral data | Browsing patterns, login times | MEDIUM |
| Device identifiers | IP address, device ID, cookies | MEDIUM |
| Employment data | Employer, job title, salary | MEDIUM |
| Health data | Medical conditions, prescriptions | CRITICAL |

### Special Category Data (GDPR Article 9)

Requires the highest level of protection:
- Racial or ethnic origin
- Political opinions
- Religious or philosophical beliefs
- Trade union membership
- Genetic data
- Biometric data for identification
- Health data
- Sex life or sexual orientation
- Criminal convictions

**Most banking GenAI systems should NOT process special category data.**

## PII Detection

### Automated PII Detection

```python
import re
from typing import List, Tuple

class PIIDetector:
    """Detect PII in text data."""

    PATTERNS = {
        "ssn": re.compile(r'\b\d{3}-\d{2}-\d{4}\b'),
        "credit_card": re.compile(r'\b(?:\d[ -]*?){13,19}\b'),
        "email": re.compile(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b'),
        "phone": re.compile(r'\b(?:\+\d{1,3}[- ]?)?\(?\d{3}\)?[- ]?\d{3}[- ]?\d{4}\b'),
        "passport": re.compile(r'\b[A-Z]{1,2}\d{6,9}\b'),
        "date_of_birth": re.compile(r'\b(?:DOB|Date of Birth|Birth Date)[:\s]*(\d{1,2}[-/]\d{1,2}[-/]\d{4})\b'),
        "bank_account": re.compile(r'\b\d{8,17}\b'),  # Varies by country
        "ip_address": re.compile(r'\b(?:\d{1,3}\.){3}\d{1,3}\b'),
    }

    def detect(self, text: str) -> List[PIIMatch]:
        """Detect all PII in the text."""
        matches = []
        for pii_type, pattern in self.PATTERNS.items():
            for match in pattern.finditer(text):
                matches.append(PIIMatch(
                    pii_type=pii_type,
                    value=match.group(),
                    start=match.start(),
                    end=match.end(),
                    confidence=self.assess_confidence(pii_type, match.group()),
                ))
        return matches

    def assess_confidence(self, pii_type: str, value: str) -> float:
        """Assess confidence that a match is actual PII."""
        # Checksum validation for SSN, credit cards, etc.
        if pii_type == "ssn":
            return 1.0 if self.valid_ssn_format(value) else 0.5
        if pii_type == "credit_card":
            digits = re.sub(r'[\s-]', '', value)
            return 1.0 if self.luhn_check(digits) else 0.3
        return 0.8  # Default confidence
```

### PII Detection in AI Inputs

```python
class AIInputPIIHandler:
    """Handle PII in inputs to AI systems."""

    def __init__(self, detector: PIIDetector, redactor: PIIRedactor):
        self.detector = detector
        self.redactor = redactor
        self.audit_logger = AuditLogger()

    def process_input(self, input_text: str, context: InputContext) -> ProcessedInput:
        """Process input text, detect PII, and decide how to handle it."""
        # Detect PII
        pii_matches = self.detector.detect(input_text)

        # Log detection (not the PII values)
        self.audit_logger.log_detection(
            pii_types=[m.pii_type for m in pii_matches],
            count=len(pii_matches),
            context=context,
        )

        # Decision based on PII type and context
        if self.contains_special_category(pii_matches):
            return ProcessedInput(
                allowed=False,
                reason="Special category data detected",
                redacted_text=self.redactor.redact_all(input_text, pii_matches),
                requires_explicit_consent=True,
            )

        if self.contains_critical_pii(pii_matches):
            return ProcessedInput(
                allowed=False,
                reason="Critical PII detected (SSN, card number, etc.)",
                redacted_text=self.redactor.redact_all(input_text, pii_matches),
                requires_escalation=True,
            )

        if pii_matches:
            # High/medium PII detected -- redact before sending to AI
            return ProcessedInput(
                allowed=True,
                redacted_text=self.redactor.redact_all(input_text, pii_matches),
                pii_detected=True,
                pii_types=[m.pii_type for m in pii_matches],
            )

        return ProcessedInput(allowed=True, redacted_text=input_text, pii_detected=False)
```

## PII Handling Strategies

### 1. Avoidance

Do not collect or process PII at all.

```
AVOIDANCE EXAMPLES:
├── Use customer ID instead of name in AI prompts
├── Hash email addresses before storing
├── Use aggregated statistics instead of individual records
├── Tokenize account numbers before any processing
└── Use synthetic data for testing
```

### 2. Minimization

Collect and process only the minimum PII necessary.

```
MINIMIZATION EXAMPLES:
├── Collect only last 4 digits of card number (not full PAN)
├── Use age range instead of date of birth
├── Use ZIP code instead of full address
├── Collect only data elements needed for the specific AI task
└── Delete PII as soon as it is no longer needed
```

### 3. Anonymization

Remove all identifying information so data cannot be linked to an individual.

```
ANONYMIZATION TECHNIQUES:
├── Suppression: Remove identifiers entirely
├── Generalization: Replace specific values with ranges
│   ├── Age 35 -> Age range 30-40
│   ├── Salary $75,432 -> Salary range $70K-$80K
│   └── ZIP 10001 -> ZIP 100XX
├── Perturbation: Add noise to values
│   ├── Transaction amount + random offset
│   └── Location coordinates + random displacement
├── Aggregation: Use summary statistics
│   ├── Average transaction amount by region
│   └── Count of transactions by month
└── k-Anonymity: Each record is indistinguishable from at least k-1 others
```

```python
def anonymize_record(record: dict, k: int = 5) -> dict:
    """Anonymize a record to achieve k-anonymity."""
    anonymized = record.copy()

    # Generalize age
    if "age" in anonymized:
        anonymized["age_range"] = f"{(anonymized['age'] // 10) * 10}-{(anonymized['age'] // 10) * 10 + 9}"
        del anonymized["age"]

    # Generalize ZIP
    if "zip_code" in anonymized:
        anonymized["zip_code"] = anonymized["zip_code"][:3] + "XX"

    # Generalize income
    if "income" in anonymized:
        anonymized["income_range"] = f"${(anonymized['income'] // 10000) * 10}K-${(anonymized['income'] // 10000) * 10 + 10}K"
        del anonymized["income"]

    # Remove direct identifiers
    for field in ["name", "email", "phone", "ssn", "account_number"]:
        if field in anonymized:
            del anonymized[field]

    return anonymized
```

### 4. Pseudonymization

Replace identifiers with reversible pseudonyms.

```
PSEUDONYMIZATION:
├── Replace names with consistent pseudonyms
├── Replace account numbers with tokens
├── Use format-preserving encryption (FPE)
├── Maintain mapping in secure vault (separate from data)
└── Can be reversed with access to mapping
```

```python
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms
from cryptography.hazmat.backends import default_backend

class Pseudonymizer:
    """Pseudonymize PII using format-preserving encryption."""

    def __init__(self, key: bytes, tweak: bytes):
        # FF3-1 algorithm for format-preserving encryption
        self.cipher = Cipher(algorithms.AES(key), mode=None, backend=default_backend())
        self.tweak = tweak

    def pseudonymize(self, value: str, domain: str) -> str:
        """Create a pseudonym for a value."""
        # Domain determines the format (e.g., "email", "ssn")
        encrypted = self.encrypt_with_fpe(value, domain)
        return encrypted

    def depseudonymize(self, pseudonym: str, domain: str) -> str:
        """Recover the original value from a pseudonym."""
        return self.decrypt_with_fpe(pseudonym, domain)
```

### 5. Encryption

Protect PII using cryptographic encryption.

```
ENCRYPTION STANDARDS:
├── Data at rest: AES-256-GCM
├── Data in transit: TLS 1.2+ (TLS 1.3 preferred)
├── Key management: HSM or cloud KMS
├── Key rotation: Annual minimum
├── Separate encryption keys from encrypted data
├── Use different keys for different data classifications
└── Document all cryptographic key usage
```

## PII in Non-Production Environments

### NEVER Use Production PII in Non-Production

```
NON-PRODUCTION DATA RULES:
├── Development: Use synthetic data ONLY
├── Testing: Use synthetic data or properly masked production data
├── Staging: Use masked production data (if necessary) or synthetic data
├── Training: Never use production data for ML training without explicit authorization
└── Disaster recovery: Encrypted production data, access-restricted
```

### Data Masking for Non-Production

```python
class ProductionDataMasker:
    """Mask production data for use in non-production environments."""

    def mask_dataset(self, df: DataFrame, environment: str) -> DataFrame:
        """Apply masking rules based on target environment."""
        masking_rules = {
            "development": {
                "name": "synthetic_name",
                "email": "fake@synthetic.com",
                "phone": "***-***-****",
                "ssn": "***-**-****",
                "pan": "****-****-****-****",
                "address": "123 Synthetic St, City, ST 12345",
            },
            "testing": {
                "name": "pseudonymize",
                "email": "hash_and_domain",
                "phone": "preserve_format_mask_digits",
                "ssn": "***-**-XXXX",
                "pan": "****-****-****-XXXX",
                "address": "generalize_to_city",
            },
            "staging": {
                "name": "pseudonymize",
                "email": "hash_and_domain",
                "phone": "preserve_format_mask_digits",
                "ssn": "***-**-XXXX",
                "pan": "****-****-****-XXXX",
                "address": "generalize_to_city",
            },
        }

        rules = masking_rules[environment]
        masked = df.copy()

        for column, rule in rules.items():
            if column in masked.columns:
                masked[column] = self.apply_rule(masked[column], rule)

        return masked
```

## PII Access Controls

```
PII ACCESS MATRIX:
┌─────────────────┬───────────┬───────────┬───────────┬──────────┐
│ Role            │ View Raw  │ View      │ View      │ Export   │
│                 │ PII       │ Masked    │ Aggregated│          │
├─────────────────┼───────────┼───────────┼───────────┼──────────┤
│ Customer Service│ No*       │ Yes       │ Yes       │ No       │
│ Fraud Analyst   │ Yes**     │ Yes       │ Yes       │ Yes**    │
│ AI Engineer     │ No        │ No        │ Yes       │ No       │
│ Data Scientist  │ No***     │ Yes       │ Yes       │ Yes***   │
│ Auditor         │ No        │ Yes       │ Yes       │ Yes      │
│ Compliance      │ Yes       │ Yes       │ Yes       │ Yes      │
└─────────────────┴───────────┴───────────┴───────────┴──────────┘

* With customer consent and business justification
** With audit trail and specific case authorization
*** With approved data request and masked data preferred
```

## PII Breach Response

```
PII BREACH RESPONSE:
├── Detect: Automated monitoring identifies potential breach
├── Contain: Isolate affected systems, prevent further exposure
├── Assess:
│   ├── What PII was exposed?
│   ├── How many individuals affected?
│   ├── How did the breach occur?
│   └── What is the risk to individuals?
├── Notify:
│   ├── Supervisory authority (GDPR: within 72 hours)
│   ├── Affected individuals (if high risk)
│   ├── Law enforcement (if criminal activity suspected)
│   └── Regulators (per banking requirements)
├── Remediate: Fix the root cause, strengthen controls
├── Document: Complete incident record with timeline and actions
└── Review: Post-incident review, lessons learned, improvements
```

## Common Interview Questions

### Question 1: "How do you handle PII in prompts sent to an external AI API?"

**Good answer structure**:
I would implement a multi-layer approach: (1) Detect PII in prompts before they are sent using automated detection (regex, NER models). (2) Redact or replace detected PII before the prompt leaves the bank's environment. (3) Use customer IDs instead of names, tokenized account numbers instead of raw numbers. (4) Log the PII detection and redaction actions (not the PII itself). (5) If PII cannot be redacted without breaking the prompt's functionality, the prompt should not be sent to the external API -- use an internal model or alternative approach instead.

### Question 2: "Is a vector embedding of customer data considered PII?"

**Good answer**:
Yes, potentially. If the embedding is derived from personal data, it encodes information about that individual and should be treated as PII. The challenge is that you cannot "read" PII from an embedding, but you can potentially use it to identify or profile the individual. Therefore: (1) Hash customer identifiers before embedding. (2) Implement access controls on vector databases. (3) Include vector embeddings in DSAR fulfillment. (4) Delete embeddings when source data is deleted. (5) Document the data lineage from source to embedding.

### Question 3: "What's the difference between anonymization and pseudonymization?"

**Good answer**:
Anonymization is irreversible -- once data is anonymized, it cannot be linked back to an individual. Pseudonymization is reversible -- identifiers are replaced with pseudonyms, and the mapping is stored securely. Under GDPR, anonymized data is no longer personal data (GDPR doesn't apply), but pseudonymized data IS still personal data (GDPR still applies). Anonymization is preferred for analytics and ML training, while pseudonymization is useful when you need to re-identify data for specific purposes (customer support, fraud investigation).
