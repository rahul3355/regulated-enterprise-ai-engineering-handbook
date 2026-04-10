# Glossary: How to Use

## Purpose

This glossary provides definitions for terms used throughout the Banking GenAI Engineering Academy. It covers terms from multiple domains: banking, security, compliance, AI/ML, Kubernetes, backend engineering, frontend engineering, DevOps, databases, and system design.

## How to Use This Glossary

### For Learning

1. **Start with your domain**: If you are a backend engineer, start with [Backend Terms](backend-terms.md), then explore adjacent domains.
2. **Learn related terms together**: When you look up one term, note the "Related Concepts" for adjacent terms to learn.
3. **Note common misunderstandings**: These highlight frequently confused concepts.

### For Reference

1. **Search for the term**: Use your editor's search to find a specific term.
2. **Check the example**: Every term includes a concrete example in banking context.
3. **Cross-reference**: Related terms link to each other for deeper understanding.

### For Interview Preparation

1. **Review key terms**: Focus on terms marked as "Commonly asked in interviews."
2. **Understand, do not memorize**: Interviewers want to hear your reasoning, not your definition.
3. **Connect to experience**: Be ready to discuss how you have used or encountered each term.

## Glossary Files

| File | Domain | Key Terms |
|------|--------|-----------|
| [banking-terms.md](banking-terms.md) | Banking and Finance | AML, KYC, Basel III, LTV, DTI |
| [security-terms.md](security-terms.md) | Information Security | Zero Trust, CVE, OAuth, PII, PCI-DSS |
| [compliance-terms.md](compliance-terms.md) | Regulatory Compliance | GLBA, SOX, ECOA, TILA, GDPR |
| [ai-terms.md](ai-terms.md) | AI/ML/GenAI | RAG, Fine-tuning, Hallucination, Embedding, Token |
| [kubernetes-terms.md](kubernetes-terms.md) | Kubernetes | Pod, Deployment, Service, Ingress, HPA |
| [backend-terms.md](backend-terms.md) | Backend Engineering | API, Microservice, Idempotency, Circuit Breaker |
| [frontend-terms.md](frontend-terms.md) | Frontend Engineering | Web Vitals, SSR, CSR, PWA, RUM |
| [devops-terms.md](devops-terms.md) | DevOps and CI/CD | CI/CD, IaC, Blue-Green, Canary, GitOps |
| [database-terms.md](database-terms.md) | Databases | ACID, CAP, Index, Sharding, Replication |
| [system-design-terms.md](system-design-terms.md) | System Design | Load Balancer, CDN, Consistent Hashing, Backpressure |

## Term Entry Format

Each term entry follows this structure:

```
### TERM NAME

**Definition**: Clear, concise definition of the term.

**Banking Example**: Concrete example in a banking context.

**Related Concepts**: Other terms you should know alongside this one.

**Common Misunderstanding**: A frequently held incorrect belief about this term.

**Interview Relevance**: Whether this term is commonly discussed in technical interviews.
```

## Example Entry

### Idempotency

**Definition**: A property of an operation whereby performing the operation multiple times produces the same result as performing it once.

**Banking Example**: A mortgage application submission API must be idempotent. If the customer clicks "Submit" twice due to a slow network, only one application should be created. The API uses a client-generated idempotency key to detect and deduplicate duplicate submissions.

**Related Concepts**: Retry, Exactly-once semantics, Deduplication, REST

**Common Misunderstanding**: "Idempotent means the response is always the same." Not true. The first call might create a resource (201 Created), while subsequent calls return the existing resource (200 OK). The side effect is the same (one resource exists), but the response may differ.

**Interview Relevance**: HIGH. Frequently asked in system design interviews for payments and banking systems.

## Contributing

If you find a term missing or a definition unclear, submit an update to this glossary. Every engineer is responsible for keeping our shared vocabulary current.
