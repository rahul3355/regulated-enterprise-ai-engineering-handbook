# Hallucination Reduction

## What Is Hallucination in RAG?

Hallucination occurs when the LLM generates information that is not supported by the retrieved context. In banking, hallucinations can lead to:
- Incorrect financial advice to customers
- Regulatory compliance violations
- Reputational damage
- Legal liability

**Example**:
- Context: "Processing fee is 1.5% of loan amount"
- Hallucinated response: "Processing fee is 2.5% of loan amount" (WRONG)
- Grounded response: "Processing fee is 1.5% of loan amount [Source: Policy v3.2]" (CORRECT)

## Hallucination Rates by Configuration

| Configuration | Hallucination Rate | Notes |
|---|---|---|
| No context (pure generation) | 30-50% | Model makes things up |
| RAG, no constraints | 15-25% | Context helps but model adds details |
| RAG, grounded prompt | 8-15% | Explicit instructions help |
| RAG, grounded + citations | 5-10% | Citations force accountability |
| RAG, grounded + citations + verification | 2-5% | Post-generation check catches errors |
| RAG, all techniques + human review | < 2% | Production target for banking |

## Techniques for Reducing Hallucinations

### 1. Grounded System Prompts

```python
GROUNDED_SYSTEM_PROMPT = """You are a banking policy assistant. Your role is to answer questions 
based on the provided policy documents.

CRITICAL RULES:
1. Answer ONLY using information from the provided context documents.
2. Do NOT use any information from your general training data.
3. If the context does not contain the answer, say: "I don't have enough information to answer this question based on the available documents."
4. Do NOT make up numbers, percentages, dates, or policy details.
5. Do NOT infer or guess at information not explicitly stated in the context.
6. Every factual claim must be followed by a citation to the source document.
7. If different sources conflict, state both positions and cite each source.
8. If you are uncertain, say so rather than guessing.

Your responses must be accurate, complete, and fully supported by the provided documents."""
```

### 2. Context Format with Source Boundaries

```
[BEGIN DOCUMENT: Personal Loan Policy v3.2 - Section 4.1]
The processing fee for personal loans is calculated as 1.5% of the 
principal loan amount. The minimum processing fee is $500 and the 
maximum is $5,000. This fee is non-refundable.
[END DOCUMENT]

[BEGIN DOCUMENT: Personal Loan Policy v3.2 - Section 5.3]
Late payment penalty is 2% of the overdue EMI amount or $25, whichever 
is greater. The penalty is applied after a grace period of 15 days from 
the due date.
[END DOCUMENT]
```

Clear boundaries help the model distinguish provided context from its own knowledge.

### 3. Temperature and Generation Settings

```python
llm = ChatOpenAI(
    model="gpt-4o-mini",
    temperature=0.0,      # Deterministic output
    top_p=0.1,            # Very narrow sampling
    frequency_penalty=0,  # No penalty for repetition
    presence_penalty=0,   # No penalty for topic persistence
)
```

**Banking recommendation**: temperature=0.0 for all policy Q&A. For creative tasks (drafting emails), temperature=0.3-0.5 is acceptable.

### 4. Constrained Generation with Output Verification

```python
def verify_response(response: str, context: str) -> dict:
    """Check that the response is grounded in context."""
    
    # Extract claims from the response
    claims = extract_claims(response)
    
    # Check each claim against context
    verified = 0
    unverified = 0
    hallucinated = []
    
    for claim in claims:
        if claim_in_context(claim, context):
            verified += 1
        else:
            unverified += 1
            hallucinated.append(claim)
    
    total_claims = verified + unverified
    hallucination_rate = unverified / total_claims if total_claims > 0 else 0
    
    return {
        "verified": verified,
        "unverified": unverified,
        "hallucination_rate": hallucination_rate,
        "hallucinated_claims": hallucinated,
        "acceptable": hallucination_rate < 0.1  # < 10% threshold
    }

def claim_in_context(claim: str, context: str) -> bool:
    """Check if a claim is supported by the context."""
    # Simple approach: check for key term overlap
    claim_terms = set(extract_ngrams(claim, n=2))
    context_terms = set(extract_ngrams(context, n=2))
    
    overlap = len(claim_terms & context_terms)
    return overlap / len(claim_terms) > 0.5 if claim_terms else True

def claim_in_context_nli(claim: str, context: str, nli_model) -> bool:
    """Use NLI model for proper entailment checking."""
    # More accurate: does the context entail the claim?
    result = nli_model.predict(context, claim)
    return result == "entailment"
```

### 5. Self-Consistency Check

Ask the model to verify its own answer.

```python
def self_consistency_check(query: str, response: str, context: str, llm) -> dict:
    """Ask the model to fact-check its own response."""
    
    prompt = f"""Review the following answer to the question and check for accuracy.

Question: {query}

Context: {context[:2000]}...

Answer: {response}

Check:
1. Is every factual claim in the answer supported by the context?
2. Are there any numbers, dates, or policy details not found in the context?
3. Does the answer contradict anything in the context?

For each issue found, describe it specifically. If no issues, say "No issues found."
"""
    
    review = llm.generate(prompt)
    
    has_issues = "no issues" not in review.lower()
    
    return {
        "has_issues": has_issues,
        "review": review,
        "needs_revision": has_issues
    }
```

### 6. Citation Enforcement

Force the model to cite sources, making hallucination detectable.

```python
CITATION_ENFORCED_PROMPT = """Answer the question using ONLY the provided context.

For EVERY factual statement, include a citation like this:
"The processing fee is 1.5% [Source: Personal Loan Policy, Section 4.1]"

If the context doesn't contain the answer, respond with:
"I don't have enough information in the provided documents to answer this question."

Rules:
- No citations = hallucination (will be rejected)
- Fabricated citations = hallucination (will be rejected)
- Every number/percentage must have a citation

Context:
{context}

Question: {question}"""
```

### 7. Retrieval Quality Gate

Before generating, check if the retrieved documents are actually relevant.

```python
def retrieval_quality_gate(query: str, retrieved_docs: list, reranker) -> bool:
    """Check if retrieved documents are relevant enough to answer the query."""
    
    if not retrieved_docs:
        return False
    
    # Use re-ranker scores as relevance proxy
    doc_texts = [doc.page_content for doc in retrieved_docs]
    ranked = reranker.rerank(query, doc_texts, top_k=1)
    
    top_score = ranked[0]["score"] if ranked else 0
    
    # If the best document scores below threshold, don't generate
    return top_score > 0.3  # Threshold depends on re-ranker
```

### 8. Multi-Source Consistency Check

```python
def check_source_consistency(docs: list) -> dict:
    """Check if retrieved documents contain consistent information."""
    
    # Extract key facts from each document
    facts_by_source = {}
    for doc in docs:
        facts = extract_key_facts(doc.page_content)
        facts_by_source[doc.metadata["doc_title"]] = facts
    
    # Find conflicting facts
    all_facts = {}
    for source, facts in facts_by_source.items():
        for fact in facts:
            if fact not in all_facts:
                all_facts[fact] = []
            all_facts[fact].append(source)
    
    # Facts mentioned in multiple sources with different values
    conflicts = []
    for fact, sources in all_facts.items():
        if len(sources) > 1:
            # Check if the actual values differ
            values = get_fact_values(fact, facts_by_source)
            if len(set(values)) > 1:
                conflicts.append({
                    "fact": fact,
                    "sources": sources,
                    "values": values
                })
    
    return {
        "consistent": len(conflicts) == 0,
        "conflicts": conflicts,
        "source_count": len(facts_by_source)
    }
```

### 9. Structured Output for Factual Answers

For specific factual questions, use structured output instead of free text.

```python
from pydantic import BaseModel, Field

class FactualAnswer(BaseModel):
    """Structured answer for factual banking questions."""
    question: str = Field(description="The original question")
    answer: str = Field(description="The answer, grounded in context")
    confidence: str = Field(description="high/medium/low")
    sources: list[str] = Field(description="List of source document titles")
    key_facts: list[str] = Field(description="Individual factual claims, each verified")
    caveats: list[str] = Field(description="Any limitations or conditions")
    cannot_answer: bool = Field(description="True if context doesn't support an answer")
    reason_if_cannot_answer: str = Field(description="Why the question cannot be answered")

def generate_structured_answer(query: str, context: str, llm) -> FactualAnswer:
    """Generate a structured, verifiable answer."""
    
    prompt = f"""Extract the answer from the context below in the specified format.

Context:
{context}

Question: {query}"""
    
    # Use structured output (OpenAI supports this natively)
    return llm.invoke(prompt, response_format=FactualAnswer)
```

### 10. Human-in-the-Loop for Low Confidence

```python
def handle_low_confidence(query: str, response: str, confidence_score: float,
                          user_id: str) -> str:
    """Route low-confidence responses for human review."""
    
    if confidence_score < 0.5:
        # Flag for human review
        review_ticket = create_review_ticket(
            query=query,
            response=response,
            user_id=user_id,
            reason=f"Low confidence score: {confidence_score:.2f}",
            priority="high" if confidence_score < 0.3 else "medium"
        )
        
        return (
            "I've found some relevant information but I'm not fully confident "
            "in providing a complete answer. I've flagged this for review by "
            "a banking specialist who will follow up."
        )
    
    return response
```

## Hallucination Detection Pipeline

```python
class HallucinationDetector:
    def __init__(self, nli_model, llm_for_review):
        self.nli_model = nli_model
        self.llm_for_review = llm_for_review
    
    def detect(self, query: str, context: str, response: str) -> dict:
        """Full hallucination detection pipeline."""
        
        results = {
            "query": query,
            "hallucinations": [],
            "severity": "none",
            "should_block": False
        }
        
        # Step 1: Extract claims
        claims = self._extract_claims(response)
        
        # Step 2: NLI-based verification
        for claim in claims:
            entailment = self.nli_model.predict(context, claim)
            if entailment != "entailment":
                results["hallucinations"].append({
                    "claim": claim,
                    "type": "not_entailed" if entailment == "neutral" else "contradiction",
                    "severity": "high" if entailment == "contradiction" else "medium"
                })
        
        # Step 3: LLM self-review
        review = self._self_review(query, context, response)
        if review.get("has_issues"):
            results["llm_review"] = review
        
        # Step 4: Check citation coverage
        citation_rate = self._citation_rate(response)
        if citation_rate < 0.5:
            results["hallucinations"].append({
                "type": "low_citation_rate",
                "value": citation_rate,
                "severity": "medium"
            })
        
        # Step 5: Determine overall severity
        if any(h["severity"] == "high" for h in results["hallucinations"]):
            results["severity"] = "high"
            results["should_block"] = True
        elif len(results["hallucinations"]) > 2:
            results["severity"] = "medium"
            results["should_block"] = False
        elif results["hallucinations"]:
            results["severity"] = "low"
        
        return results
    
    def _extract_claims(self, text: str) -> list[str]:
        """Extract factual claims from text."""
        # Split into sentences
        sentences = sent_tokenize(text)
        
        # Filter out non-claim sentences
        claims = []
        for s in sentences:
            # Skip: questions, meta-statements, hedging
            if any(x in s.lower() for x in ["i don't know", "i cannot", 
                                               "not available", "unknown"]):
                continue
            # Skip if too short
            if len(s.split()) < 4:
                continue
            claims.append(s)
        
        return claims
    
    def _citation_rate(self, text: str) -> float:
        """Calculate what fraction of sentences have citations."""
        sentences = sent_tokenize(text)
        if not sentences:
            return 1.0
        
        cited = sum(1 for s in sentences if re.search(r'\[\d+\]|\[Source:', s))
        return cited / len(sentences)
```

## Banking-Specific Hallucination Risks

| Risk Area | Example | Impact | Mitigation |
|---|---|---|---|
| **Numbers** | Wrong fee %, interest rate | High customer impact | Structured extraction; verify numbers |
| **Dates** | Wrong effective date for policy | Compliance violation | Date validation against source |
| **Regulation names** | Citing wrong regulation | Legal liability | Regulation name verification |
| **Eligibility criteria** | Wrong requirements | Bad customer advice | Strict context-only generation |
| **Process steps** | Missing or wrong steps | Operational errors | Numbered list verification |

## Best Practices Summary

1. **Temperature = 0**: For all factual Q&A, use deterministic generation
2. **Explicit "I don't know" instruction**: Tell the model to say it doesn't know
3. **Source boundaries**: Use clear `[BEGIN DOC]` / `[END DOC]` markers
4. **Citation enforcement**: Require and verify citations
5. **NLI verification**: Post-generate claim verification with NLI model
6. **Self-consistency**: Ask the model to review its own answer
7. **Retrieval quality gate**: Don't generate if retrieval scores are low
8. **Human review**: Route low-confidence responses for human review
9. **Monitor hallucination rate**: Track as a key production metric
10. **Golden dataset testing**: Regularly test against known-answer questions
