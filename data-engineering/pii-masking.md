# PII Masking and Tokenization in Data Pipelines

## Overview

Personally Identifiable Information (PII) in banking data pipelines must be protected at every stage: ingestion, transformation, storage, and consumption. Masking and tokenization are the primary techniques for protecting PII while maintaining data utility for analytics and GenAI workloads. This guide covers practical PII detection, masking strategies, tokenization patterns, and compliance requirements.

## PII Detection

```python
"""
PII detection in banking data pipelines.
Identifies and classifies sensitive fields for masking.
"""
import re
from typing import Dict, List, Optional
from dataclasses import dataclass

@dataclass
class PIIFinding:
    field_name: str
    pii_type: str
    confidence: float
    value_hash: str  # Hash for dedup, not raw value

class PIIDetector:
    """Detect PII in data records."""
    
    # PII patterns for detection
    PATTERNS = {
        'SSN': [
            re.compile(r'\b\d{3}-\d{2}-\d{4}\b'),
            re.compile(r'\b\d{9}\b'),  # SSN without dashes
        ],
        'CREDIT_CARD': [
            re.compile(r'\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b'),
            re.compile(r'\b\d{13,19}\b'),
        ],
        'ACCOUNT_NUMBER': [
            re.compile(r'\b\d{10,16}\b'),
        ],
        'EMAIL': [
            re.compile(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b'),
        ],
        'PHONE': [
            re.compile(r'\b\+?1?[\s-]?\(?\d{3}\)?[\s-]?\d{3}[\s-]?\d{4}\b'),
        ],
        'ROUTING_NUMBER': [
            re.compile(r'\b\d{9}\b'),
        ],
    }
    
    # Column name indicators
    PII_COLUMNS = {
        'ssn', 'social_security', 'national_id',
        'account_number', 'acct_no', 'card_number', 'credit_card',
        'email', 'email_address',
        'phone', 'phone_number', 'mobile',
        'date_of_birth', 'dob', 'birth_date',
        'address', 'street', 'zip_code',
        'password', 'pin', 'cvv', 'cvc',
    }
    
    def detect_in_record(self, record: Dict) -> List[PIIFinding]:
        """Scan a record for PII."""
        findings = []
        
        for field, value in record.items():
            if value is None:
                continue
            
            value_str = str(value)
            
            # Check column name
            if field.lower() in self.PII_COLUMNS:
                findings.append(PIIFinding(
                    field_name=field,
                    pii_type=self._classify_column(field),
                    confidence=0.95,
                    value_hash=hash(value_str),
                ))
                continue
            
            # Check value patterns
            for pii_type, patterns in self.PATTERNS.items():
                for pattern in patterns:
                    if pattern.search(value_str):
                        findings.append(PIIFinding(
                            field_name=field,
                            pii_type=pii_type,
                            confidence=0.8,
                            value_hash=hash(value_str),
                        ))
                        break
        
        return findings
    
    def _classify_column(self, column: str) -> str:
        """Classify PII type based on column name."""
        col_lower = column.lower()
        if 'ssn' in col_lower or 'social' in col_lower:
            return 'SSN'
        if 'card' in col_lower or 'credit' in col_lower:
            return 'CREDIT_CARD'
        if 'account' in col_lower or 'acct' in col_lower:
            return 'ACCOUNT_NUMBER'
        if 'email' in col_lower:
            return 'EMAIL'
        if 'phone' in col_lower or 'mobile' in col_lower:
            return 'PHONE'
        if 'dob' in col_lower or 'birth' in col_lower:
            return 'DATE_OF_BIRTH'
        return 'UNKNOWN_PII'
```

## Masking Strategies

```python
"""
PII masking functions for data pipelines.
Different strategies for different use cases.
"""
import hashlib
import hmac
from typing import Optional

class PIIMasker:
    """Apply masking transformations to PII fields."""
    
    def __init__(self, encryption_key: str):
        self.encryption_key = encryption_key.encode()
    
    def mask_ssn(self, ssn: str) -> str:
        """Mask SSN: show only last 4 digits."""
        if not ssn:
            return None
        clean = re.sub(r'[\s-]', '', ssn)
        if len(clean) == 9:
            return f"***-**-{clean[-4:]}"
        return "[SSN_REDACTED]"
    
    def mask_account_number(self, account: str) -> str:
        """Mask account number: show only last 4 digits."""
        if not account:
            return None
        if len(account) >= 4:
            return '*' * (len(account) - 4) + account[-4:]
        return "[ACCOUNT_REDACTED]"
    
    def mask_email(self, email: str) -> str:
        """Mask email: keep domain, obscure local part."""
        if not email:
            return None
        try:
            local, domain = email.rsplit('@', 1)
            masked_local = local[0] + '*' * (len(local) - 1)
            return f"{masked_local}@{domain}"
        except ValueError:
            return "[EMAIL_REDACTED]"
    
    def mask_name(self, name: str) -> str:
        """Mask name: keep first letter, rest asterisks."""
        if not name:
            return None
        parts = name.split()
        return ' '.join(p[0] + '*' * (len(p) - 1) if len(p) > 1 else p for p in parts)
    
    def tokenize(self, value: str, token_type: str = 'default') -> str:
        """
        Deterministic tokenization: same value -> same token.
        Enables joining on tokenized fields across datasets.
        """
        if not value:
            return None
        
        token = hmac.new(
            self.encryption_key,
            f"{token_type}:{value}".encode(),
            hashlib.sha256
        ).hexdigest()[:16]
        
        return f"tok_{token_type}_{token}"
    
    def hash_with_salt(self, value: str) -> str:
        """One-way hash for irreversible anonymization."""
        if not value:
            return None
        return hashlib.sha256(
            f"{self.encryption_key.decode()}:{value}".encode()
        ).hexdigest()
    
    def apply_masking_pipeline(self, record: Dict) -> Dict:
        """Apply full masking pipeline to a record."""
        masked = record.copy()
        
        # SSN: redact all but last 4
        if 'ssn' in masked:
            masked['ssn_masked'] = self.mask_ssn(masked['ssn'])
            del masked['ssn']
        
        # Account number: tokenized for joining, masked for display
        if 'account_number' in masked:
            masked['account_token'] = self.tokenize(masked['account_number'], 'account')
            masked['account_display'] = self.mask_account_number(masked['account_number'])
            del masked['account_number']
        
        # Email: tokenized for analytics
        if 'email' in masked:
            masked['email_token'] = self.tokenize(masked['email'], 'email')
            masked['email_masked'] = self.mask_email(masked['email'])
            del masked['email']
        
        # Name: hash for anonymization
        if 'first_name' in masked:
            masked['name_hash'] = self.hash_with_salt(
                f"{masked.get('first_name', '')} {masked.get('last_name', '')}"
            )
            del masked['first_name']
            del masked['last_name']
        
        return masked
```

## Tokenization for Analytics

```sql
-- Tokenization in Postgres: enable analytics on PII without exposing raw values
-- Tokens are deterministic, enabling joins and groupings

CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Tokenization function
CREATE OR REPLACE FUNCTION tokenize_pii(
    raw_value TEXT,
    token_type TEXT,
    secret_key TEXT DEFAULT 'change-me-in-production'
) RETURNS TEXT AS $$
BEGIN
    RETURN 'tok_' || token_type || '_' || 
           SUBSTR(encode(hmac(token_type || ':' || raw_value, secret_key, 'sha256'), 'hex'), 1, 16);
END;
$$ LANGUAGE plpgsql IMMUTABLE;

-- Create tokenized columns
ALTER TABLE analytics_customers ADD COLUMN email_token TEXT;
ALTER TABLE analytics_customers ADD COLUMN ssn_token TEXT;
ALTER TABLE analytics_customers ADD COLUMN account_token TEXT;

-- Populate tokens
UPDATE analytics_customers SET
    email_token = tokenize_pii(email, 'email'),
    ssn_token = tokenize_pii(ssn, 'ssn'),
    account_token = tokenize_pii(account_number, 'account');

-- Now analytics can join on tokens without seeing raw PII
SELECT 
    t.email_token,
    COUNT(*) AS txn_count,
    SUM(t.amount) AS total_spend
FROM analytics_transactions t
JOIN analytics_customers c ON t.email_token = c.email_token
GROUP BY t.email_token;
```

## GenAI-Specific PII Handling

```python
"""
PII handling in GenAI pipelines:
1. Remove PII from documents before embedding
2. Remove PII from user prompts before sending to LLM
3. Remove PII from model responses before returning to user
"""
import re

class GenAIPipelinePIIHandler:
    """Handle PII in GenAI workflows."""
    
    def clean_document_for_embedding(self, content: str) -> str:
        """Remove PII from documents before embedding."""
        # Mask account numbers
        content = re.sub(r'\b\d{10,16}\b', '[ACCOUNT_NUMBER]', content)
        
        # Mask SSNs
        content = re.sub(r'\b\d{3}-\d{2}-\d{4}\b', '[SSN]', content)
        
        # Mask credit cards
        content = re.sub(r'\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b', 
                        '[CREDIT_CARD]', content)
        
        # Mask emails
        content = re.sub(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b',
                        '[EMAIL]', content)
        
        # Mask phone numbers
        content = re.sub(r'\b\+?1?[\s-]?\(?\d{3}\)?[\s-]?\d{3}[\s-]?\d{4}\b',
                        '[PHONE]', content)
        
        # Mask dollar amounts (specific amounts might be sensitive)
        # content = re.sub(r'\$\d{1,3}(,\d{3})*(\.\d{2})?', '[AMOUNT]', content)
        
        return content
    
    def sanitize_prompt(self, prompt: str) -> tuple:
        """
        Sanitize user prompt, return (sanitized_prompt, pii_found).
        """
        pii_found = []
        sanitized = prompt
        
        # Check for PII patterns
        emails = re.findall(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b', prompt)
        if emails:
            pii_found.append({'type': 'email', 'count': len(emails)})
            for email in emails:
                sanitized = sanitized.replace(email, '[EMAIL]')
        
        account_nums = re.findall(r'\b\d{10,16}\b', prompt)
        if account_nums:
            pii_found.append({'type': 'account_number', 'count': len(account_nums)})
            for acc in account_nums:
                sanitized = sanitized.replace(acc, '[ACCOUNT_NUMBER]')
        
        return sanitized, pii_found
```

## Cross-References

- **Data Quality**: See [data-quality.md](data-quality.md) for data validation
- **Data Governance**: See [data-governance.md](data-governance.md) for compliance
- **GenAI Data Prep**: See [genai-data-prep.md](genai-data-prep.md) for document cleaning

## Interview Questions

1. **What is the difference between masking, tokenization, and hashing?**
2. **How do you enable analytics on customer data without exposing PII?**
3. **Your pipeline needs to join customer data with transaction data, but PII cannot leave the secure zone. How do you design this?**
4. **What are the compliance implications of storing PII in GenAI embeddings?**
5. **How do you detect PII in unstructured text documents?**
6. **Design a PII masking pipeline that supports both analytics and ML use cases.**

## Checklist: PII Protection

- [ ] PII detection automated at data ingestion
- [ ] Masking rules documented and approved by compliance
- [ ] Tokenization keys stored securely (KMS/HSM)
- [ ] Deterministic tokens used for joins, non-deterministic for storage
- [ ] PII removed from GenAI training documents
- [ ] Prompts sanitized before sending to LLM
- [ ] Responses checked for PII before returning to user
- [ ] Token-to-PII mapping stored separately from analytics data
- [ ] Access to raw PII restricted and audited
- [ ] PII masking tested with known test cases
- [ ] Regular audit of PII fields across all data stores
