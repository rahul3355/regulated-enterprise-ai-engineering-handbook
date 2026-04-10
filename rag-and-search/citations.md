# Citations

## Why Citations Matter

In banking, every AI response must be traceable to its source. Citations provide:
- **Auditability**: Regulators can verify where each statement came from
- **Trust**: Users can check the source documents themselves
- **Accountability**: If an answer is wrong, you can identify which source document was faulty
- **Compliance**: Many banking regulations require source attribution

## Citation Types

### Inline Citations

```
The processing fee for personal loans is 1.5% of the loan amount, 
subject to a minimum of $500 and maximum of $5,000 [1]. 
Applications must include proof of income and valid identification [2].

---
Sources:
[1] Personal Loan Policy v3.2, Section 4.1 "Fees and Charges"
[2] Personal Loan Policy v3.2, Section 3.2 "Documentation Requirements"
```

### Footnote Citations

```
The processing fee for personal loans is 1.5% of the loan amount.

¹ According to the Personal Loan Policy (v3.2, updated June 2024), 
Section 4.1: "The processing fee shall be 1.5% of the principal 
amount, with a floor of $500 and ceiling of $5,000."
```

### Block Citations

Each paragraph is followed by its source block:

```
The processing fee is 1.5% of the loan amount.
> Source: Personal Loan Policy v3.2, Section 4.1

You must provide proof of income and ID.
> Source: Personal Loan Policy v3.2, Section 3.2
```

## Implementing Citation Generation

### Prompt-Based Citation Generation

```python
CITATION_SYSTEM_PROMPT = """You are a banking policy assistant. Answer questions using ONLY the provided context. 

Rules:
1. Every factual statement must be followed by a citation in the format [Source: document_title, section]
2. If you cannot find the answer in the context, say "I don't have that information in my current knowledge base."
3. Never use information from your training data that is not in the provided context.
4. If different sources provide conflicting information, note the discrepancy.
5. Always include the document version number if available."""

CITATION_USER_PROMPT = """Context:
{context}

Question: {question}

Answer with inline citations for every factual statement."""
```

### Citation-Ready Context Assembly

```python
def assemble_citable_context(docs: list, include_metadata: bool = True) -> str:
    """Format retrieved documents with clear source markers."""
    
    context_parts = []
    for i, doc in enumerate(docs, 1):
        meta = doc.metadata
        source_info = f"Source {i}: {meta.get('doc_title', 'Unknown')}"
        
        if meta.get('doc_version'):
            source_info += f" (Version {meta['doc_version']})"
        if meta.get('section_title'):
            source_info += f" - {meta['section_title']}"
        if meta.get('updated_at'):
            source_info += f" [Updated: {meta['updated_at']}]"
        
        context_parts.append(f"[BEGIN {source_info}]\n{doc.page_content}\n[END {source_info}]")
    
    return "\n\n".join(context_parts)
```

### Post-Processing Citation Extraction

```python
def extract_citations(response: str, source_docs: list) -> dict:
    """Extract and validate citations from the response."""
    
    # Find all citation markers like [1], [2], etc.
    citation_pattern = r'\[(\d+)\]'
    cited_numbers = [int(m) for m in re.findall(citation_pattern, response)]
    
    # Map to source documents
    citations = []
    for num in cited_numbers:
        if 1 <= num <= len(source_docs):
            doc = source_docs[num - 1]
            citations.append({
                "number": num,
                "doc_title": doc.metadata.get("doc_title", "Unknown"),
                "section": doc.metadata.get("section_title", ""),
                "version": doc.metadata.get("doc_version", ""),
                "doc_id": doc.metadata.get("doc_id", ""),
                "url": doc.metadata.get("url", "")
            })
    
    return {
        "citations": citations,
        "uncited_claims": count_uncited_claims(response),
        "total_claims": count_claims(response)
    }

def validate_citations(response: str, source_docs: list) -> list[dict]:
    """Verify each citation actually supports the claim it's attached to."""
    # This is harder to automate - requires NLI model
    # For now, check that cited documents contain the key terms
    
    claims_with_citations = extract_citation_claims(response)
    validation_results = []
    
    for claim, citation_num in claims_with_citations:
        if 1 <= citation_num <= len(source_docs):
            source_doc = source_docs[citation_num - 1]
            # Simple check: does the source document contain key terms from the claim?
            key_terms = extract_key_terms(claim)
            source_lower = source_doc.page_content.lower()
            terms_found = sum(1 for term in key_terms if term.lower() in source_lower)
            
            validation_results.append({
                "claim": claim,
                "source": source_doc.metadata.get("doc_title", "Unknown"),
                "terms_match_ratio": terms_found / len(key_terms) if key_terms else 0,
                "verified": terms_found / len(key_terms) > 0.5 if key_terms else False
            })
    
    return validation_results
```

### Citation Format Templates

```python
CITATION_FORMATS = {
    "inline": "[{doc_title} v{version}, {section}]",
    "footnote": "^{doc_title} (v{version}), {section}",
    "link": "[{doc_title}]({url})",
    "full": "{doc_title} | Version {version} | {section} | Updated: {updated_at}",
    "short": "[{doc_title}]",
}

def format_citation(doc_metadata: dict, format_type: str = "inline") -> str:
    """Format a citation from document metadata."""
    template = CITATION_FORMATS.get(format_type, CITATION_FORMATS["inline"])
    return template.format(
        doc_title=doc_metadata.get("doc_title", "Unknown Document"),
        version=doc_metadata.get("doc_version", "N/A"),
        section=doc_metadata.get("section_title", ""),
        updated_at=doc_metadata.get("updated_at", "Unknown"),
        url=doc_metadata.get("url", "#")
    )
```

## Citation Quality Metrics

| Metric | Description | Target |
|---|---|---|
| **Citation rate** | % of factual statements with citations | > 90% |
| **Citation accuracy** | % of citations that point to correct source | > 95% |
| **Source diversity** | Number of distinct sources cited | 2-4 per response |
| **Recency** | Age of cited source documents | < 365 days for policies |
| **Attribution rate** | % of claims that can be traced to sources | > 85% |

### Automated Citation Quality Checker

```python
def check_citation_quality(response: str, source_docs: list) -> dict:
    """Comprehensive citation quality assessment."""
    
    # Extract citations
    citations = extract_citations(response, source_docs)
    
    # Check coverage: what % of the response is covered by cited sources?
    cited_content = ""
    for citation in citations["citations"]:
        for doc in source_docs:
            if doc.metadata.get("doc_id") == citation.get("doc_id"):
                cited_content += doc.page_content + " "
    
    # Check that key claims are supported
    claims = extract_claims(response)
    supported_claims = 0
    for claim in claims:
        if is_claim_supported(claim, cited_content):
            supported_claims += 1
    
    return {
        "citation_rate": len(citations["citations"]) / max(len(claims), 1),
        "attribution_rate": supported_claims / max(len(claims), 1),
        "source_count": len(set(c["doc_id"] for c in citations["citations"])),
        "sources": citations["citations"],
        "uncited_claims": citations.get("uncited_claims", 0)
    }
```

## Banking-Specific Citation Requirements

### Regulatory Citation Format

For regulatory compliance answers, use formal citation format:

```
Per Regulation E (12 CFR Part 1005), Section 1005.6, consumers are 
liable for no more than $50 in unauthorized electronic fund transfers 
if they notify the financial institution within 2 business days of 
learning of the loss or theft.

Reference: 12 CFR 1005.6 - Liability of consumer for unauthorized transfers
Source: Consumer Financial Protection Bureau, Regulation E
Last updated: January 2024
```

### Internal Policy Citation Format

```
According to the Personal Loan Policy document (POL-RTL-001, v3.2), 
approved by the Risk Committee on June 15, 2024:

"The processing fee for personal loans shall be calculated as 1.5% 
of the principal loan amount, subject to a minimum of $500 and a 
maximum of $5,000."

Document: Personal Loan Policy
ID: POL-RTL-001
Version: 3.2
Section: 4.1 - Fees and Charges
Approved by: Risk Committee, June 15, 2024
Effective date: July 1, 2024
```

## Citation Logging for Audit

```python
import logging
import json

audit_logger = logging.getLogger("rag.audit")

def log_citation_event(query: str, response: str, sources: list, user_id: str):
    """Log citation information for audit compliance."""
    
    audit_event = {
        "event_type": "rag_response",
        "timestamp": datetime.utcnow().isoformat(),
        "user_id": user_id,
        "query": query,
        "response_length": len(response),
        "sources": [
            {
                "doc_id": s.metadata.get("doc_id"),
                "doc_title": s.metadata.get("doc_title"),
                "doc_version": s.metadata.get("doc_version"),
                "section": s.metadata.get("section_title"),
            }
            for s in sources
        ],
        "source_count": len(sources),
    }
    
    audit_logger.info(json.dumps(audit_event))
```

## Common Citation Issues

### 1. Hallucinated Citations

LLMs may invent citation numbers or attributes.

**Mitigation**:
- Use structured prompts with explicit source blocks
- Post-process to verify all citations map to actual sources
- Reject responses with citations to non-existent sources

```python
def verify_no_hallucinated_citations(response: str, source_count: int) -> bool:
    """Check that all citation numbers are valid."""
    cited_numbers = [int(m) for m in re.findall(r'\[(\d+)\]', response)]
    return all(1 <= n <= source_count for n in cited_numbers)
```

### 2. Over-Citation

Citing the same source for every sentence is verbose and unhelpful.

**Mitigation**:
- Set a maximum citation rate (one citation per 2-3 sentences max)
- Group citations at paragraph level rather than sentence level

### 3. Under-Citation

Making claims without any citations is dangerous.

**Mitigation**:
- Require at least one citation per paragraph
- Flag responses with citation rate below threshold
- Auto-reject responses with zero citations

### 4. Stale Source Citation

Citing outdated document versions.

**Mitigation**:
- Always include document version in citation
- Flag citations to documents older than N days
- Prefer most recent document version during retrieval

```python
def check_citation_freshness(sources: list, max_age_days: int = 365) -> list[dict]:
    """Check if cited sources are current."""
    today = datetime.now()
    freshness = []
    
    for source in sources:
        updated = source.metadata.get("updated_at")
        if updated:
            age = (today - datetime.fromisoformat(updated)).days
            freshness.append({
                "doc_id": source.metadata.get("doc_id"),
                "age_days": age,
                "is_current": age < max_age_days
            })
    
    return freshness
```

## Best Practices

1. **Always use structured source blocks**: `[BEGIN Source X: Title]\nContent\n[END Source X]`
2. **Include version numbers**: Every citation should include document version
3. **Log all citations**: For audit compliance, log every source used in every response
4. **Verify citations**: Post-process to ensure citations are accurate and not hallucinated
5. **Prefer recent sources**: When multiple versions exist, cite the most recent
6. **Note discrepancies**: If sources conflict, acknowledge it explicitly
7. **Provide source links**: If documents are accessible online, include URLs
8. **Track citation quality**: Monitor citation accuracy and rate as key metrics
