# Groundedness

## What Is Groundedness?

Groundedness measures how well an AI response is supported by the provided source documents. A fully grounded response contains only information that can be traced back to the retrieved context. An ungrounded response includes claims not present in the sources.

**Groundedness is the single most important quality metric for banking RAG.** An ungrounded response may give incorrect financial advice, cite wrong regulations, or invent policy details.

## Groundedness Spectrum

```
Fully Grounded          Partially Grounded          Ungrounded (Hallucinated)
All claims backed       Some claims backed          No claims backed
by sources              by sources                  by sources
       |                        |                           |
"The fee is 1.5% [1]"    "Fee is 1.5% [1] and         "Fee is 2.5% and
                          processing takes 3 days"       takes 2 days"
                          ^backed    ^not backed         ^fabricated
```

## Measuring Groundedness

### Method 1: NLI-Based Groundedness

Natural Language Inference (NLI) models determine if a hypothesis is entailed by, neutral to, or contradicts a premise.

```python
from transformers import pipeline

nli = pipeline("text-classification", 
               model="MoritzLawor/roberta-large-mnli",
               return_all_scores=True)

def check_claim_grounded(claim: str, context: str) -> str:
    """Check if a claim is entailed by the context."""
    # NLI models take premise (context) and hypothesis (claim)
    result = nli({"text": context, "text_pair": claim})
    
    # Get the highest scoring label
    best_label = max(result, key=lambda x: x["score"])["label"]
    return best_label  # "entailment", "neutral", or "contradiction"

def compute_groundedness(response: str, context: str) -> float:
    """Compute groundedness score for a response."""
    claims = extract_claims(response)
    
    if not claims:
        return 1.0  # No claims = nothing to hallucinate
    
    grounded_count = 0
    for claim in claims:
        label = check_claim_grounded(claim, context)
        if label == "entailment":
            grounded_count += 1
        elif label == "neutral":
            grounded_count += 0.5  # Partial credit
    
    return grounded_count / len(claims)
```

### Method 2: LLM-as-Judge Groundedness

```python
def llm_groundedness_score(response: str, context: str, llm) -> dict:
    """Use LLM to evaluate groundedness."""
    
    prompt = f"""You are evaluating whether an AI response is grounded in the provided context.

CONTEXT:
{context}

RESPONSE:
{response}

Analyze each factual claim in the response and determine if it is:
- FULLY SUPPORTED: The claim is directly stated or clearly implied by the context
- PARTIALLY SUPPORTED: Some aspects of the claim are in the context but details may be added
- NOT SUPPORTED: The claim cannot be found in or contradicts the context
- META: This is not a factual claim (e.g., "I can help with that")

Respond in JSON format:
{{
  "claims": [
    {{"claim": "...", "status": "fully_supported|partially_supported|not_supported|meta"}}
  ],
  "groundedness_score": 0.0  # Fraction of factual claims that are fully supported
}}"""
    
    return parse_json(llm.generate(prompt))

# Usage
result = llm_groundedness_score(response, context, llm)
print(f"Groundedness: {result['groundedness_score']:.2f}")
for claim in result["claims"]:
    if claim["status"] == "not_supported":
        print(f"  UNGROUNDED: {claim['claim']}")
```

### Method 3: RAGAS Faithfulness

```python
from ragas.metrics import faithfulness
from ragas import evaluate
from datasets import Dataset

data = Dataset.from_dict({
    "question": ["What is the processing fee?"],
    "answer": ["The processing fee is 1.5% of the loan amount."],
    "contexts": [["The processing fee for personal loans is 1.5% of the principal."]]
})

result = evaluate(data, metrics=[faithfulness])
# faithfulness score: 0.0-1.0
```

### Method 4: Fact-Overlap Method

```python
def fact_overlap_score(response: str, context: str) -> float:
    """Measure what fraction of response key facts appear in context."""
    
    # Extract trigrams from response
    response_trigrams = set(extract_ngrams(response, n=3))
    context_trigrams = set(extract_ngrams(context, n=3))
    
    # Filter to only "factual" trigrams (containing numbers, technical terms)
    factual_trigrams = set()
    for tri in response_trigrams:
        if any(re.search(r'\d', word) for word in tri.split()):
            factual_trigrams.add(tri)
        elif any(word.lower() in BANKING_TERMS for word in tri.split()):
            factual_trigrams.add(tri)
    
    if not factual_trigrams:
        return 1.0  # No factual claims to check
    
    matched = factual_trigrams & context_trigrams
    return len(matched) / len(factual_trigrams)
```

## Groundedness Thresholds

| Score | Interpretation | Action |
|---|---|---|
| 0.9-1.0 | Excellent grounding | Accept |
| 0.7-0.9 | Good grounding | Accept, monitor |
| 0.5-0.7 | Moderate grounding | Flag for review |
| 0.3-0.5 | Poor grounding | Reject, regenerate |
| 0.0-0.3 | Very poor grounding | Block, alert |

## Groundedness in Production

### Real-Time Groundedness Check

```python
class GroundednessChecker:
    def __init__(self, method: str = "nli"):
        if method == "nli":
            self.nli = pipeline("text-classification", 
                                model="MoritzLawor/roberta-large-mnli")
        self.method = method
    
    def check(self, response: str, context: str) -> dict:
        """Check groundedness in real-time."""
        
        if self.method == "nli":
            return self._nli_check(response, context)
        else:
            return self._fact_overlap_check(response, context)
    
    def _nli_check(self, response: str, context: str) -> dict:
        claims = extract_claims(response)
        results = []
        
        for claim in claims:
            label = self._get_nli_label(claim, context)
            results.append({
                "claim": claim,
                "status": label,
                "grounded": label in ["entailment", "neutral"]
            })
        
        grounded = sum(1 for r in results if r["grounded"])
        total = len(results)
        
        return {
            "score": grounded / total if total > 0 else 1.0,
            "claims": results,
            "total_claims": total,
            "grounded_claims": grounded,
            "ungrounded_claims": total - grounded,
            "acceptable": (grounded / total if total > 0 else 1.0) >= 0.7
        }
    
    def _get_nli_label(self, claim: str, context: str) -> str:
        # Truncate context to reasonable size
        context_truncated = context[:2000]
        
        result = self.nli({"text": context_truncated, "text_pair": claim})
        return max(result, key=lambda x: x["score"])["label"]
```

### Groundedness Dashboard

Track groundedness over time to detect degradation.

```python
import matplotlib.pyplot as plt
from datetime import datetime, timedelta

def plot_groundedness_trend(groundedness_logs: list[dict]):
    """Plot groundedness score over time."""
    
    dates = [datetime.fromisoformat(log["timestamp"]) for log in groundedness_logs]
    scores = [log["groundedness_score"] for log in groundedness_logs]
    
    plt.figure(figsize=(12, 4))
    plt.plot(dates, scores, marker='o', markersize=3)
    plt.axhline(y=0.8, color='r', linestyle='--', label='Threshold (0.8)')
    plt.axhline(y=0.9, color='g', linestyle='--', label='Target (0.9)')
    plt.fill_between(dates, scores, 0.8, where=[s < 0.8 for s in scores], 
                      alpha=0.3, color='red')
    plt.title("Groundedness Score Over Time")
    plt.xlabel("Date")
    plt.ylabel("Groundedness Score")
    plt.legend()
    plt.show()
```

## Banking-Specific Groundedness Challenges

### 1. Numerical Groundedness

Banking responses often contain numbers (percentages, amounts, dates). These are high-risk for hallucination.

```python
def check_numerical_groundedness(response: str, context: str) -> dict:
    """Specifically check that all numbers in response are in context."""
    
    # Extract all numbers from response
    response_numbers = re.findall(r'\d+\.?\d*%?', response)
    
    # Check each number exists in context
    found = []
    not_found = []
    
    for num in response_numbers:
        if num in context:
            found.append(num)
        else:
            # Try close matches (within 10%)
            close_match = False
            for ctx_num in re.findall(r'\d+\.?\d*%?', context):
                try:
                    if abs(float(num.rstrip('%')) - float(ctx_num.rstrip('%'))) / float(ctx_num.rstrip('%')) < 0.1:
                        close_match = True
                        break
                except (ValueError, ZeroDivisionError):
                    pass
            
            if close_match:
                found.append(num)
            else:
                not_found.append(num)
    
    return {
        "total_numbers": len(response_numbers),
        "found_in_context": len(found),
        "not_found": len(not_found),
        "ungrounded_numbers": not_found,
        "numerical_groundedness": len(found) / len(response_numbers) if response_numbers else 1.0
    }
```

### 2. Regulatory Reference Groundedness

Check that regulation citations are accurate.

```python
REGULATION_DB = {
    "Regulation E": {"cfr": "12 CFR 1005", "topic": "Electronic Fund Transfers"},
    "Regulation Z": {"cfr": "12 CFR 1026", "topic": "Truth in Lending"},
    "Regulation B": {"cfr": "12 CFR 1002", "topic": "Equal Credit Opportunity"},
    "Regulation DD": {"cfr": "12 CFR 1030", "topic": "Truth in Savings"},
    "BSA": {"cfr": "31 USC 5311", "topic": "Bank Secrecy Act"},
}

def verify_regulation_citation(response: str) -> dict:
    """Verify that regulation citations in response are accurate."""
    
    citations = re.findall(
        r'(Regulation\s+[A-Z]|BSA|AML|Dodd-Frank|Sarbanes-Oxley)',
        response, re.IGNORECASE
    )
    
    results = []
    for citation in citations:
        citation_clean = citation.strip()
        # Check if it's a known regulation
        found = any(citation_clean.lower() in reg.lower() for reg in REGULATION_DB)
        results.append({
            "citation": citation_clean,
            "valid": found
        })
    
    return {
        "citations": results,
        "all_valid": all(r["valid"] for r in results)
    }
```

### 3. Temporal Groundedness

Policies have effective dates. A response citing an expired policy is ungrounded.

```python
def check_temporal_groundedness(response: str, source_docs: list) -> dict:
    """Check that response cites currently effective policies."""
    
    today = datetime.now()
    issues = []
    
    for doc in source_docs:
        effective_from = doc.metadata.get("effective_from")
        effective_until = doc.metadata.get("effective_until")
        
        if effective_from:
            eff_date = datetime.fromisoformat(effective_from)
            if eff_date > today:
                issues.append(f"Policy not yet effective: {doc.metadata['doc_title']}")
        
        if effective_until:
            until_date = datetime.fromisoformat(effective_until)
            if until_date < today:
                issues.append(f"Policy expired: {doc.metadata['doc_title']}")
    
    return {
        "temporally_grounded": len(issues) == 0,
        "issues": issues
    }
```

## Groundedness Improvement Strategies

### If Groundedness < 0.7:

1. **Improve retrieval**: Better retrieval means better context, which means fewer hallucinations
2. **Strengthen system prompt**: Add stronger "only use provided context" instructions
3. **Lower temperature**: Use temperature=0.0
4. **Add citation requirement**: Force model to cite sources
5. **Use smaller models**: Smaller models (gpt-4o-mini vs gpt-4o) sometimes hallucinate less because they are more constrained

### If Groundedness is 0.7-0.85:

1. **Add post-generation verification**: NLI-based claim checking
2. **Use structured output**: Force model to separate facts from reasoning
3. **Improve context assembly**: Remove irrelevant context that may confuse the model

### If Groundedness > 0.85:

1. **Monitor for drift**: Groundedness can degrade as documents change
2. **Expand golden dataset**: Test edge cases
3. **A/B test models**: Try different LLMs for incremental improvement

## Automated Groundedness Testing

```python
def batch_groundedness_test(golden_queries, pipeline, checker) -> dict:
    """Test groundedness across golden dataset."""
    
    all_scores = []
    failure_cases = []
    
    for gq in golden_queries:
        response = pipeline.query(gq.query)
        
        # Build context from retrieved docs
        context = "\n".join([d.page_content for d in response.retrieved_docs])
        
        # Check groundedness
        result = checker.check(response.text, context)
        all_scores.append(result["score"])
        
        if result["score"] < 0.7:
            failure_cases.append({
                "query": gq.query,
                "response": response.text,
                "groundedness": result["score"],
                "ungrounded_claims": [
                    c["claim"] for c in result["claims"] 
                    if not c["grounded"]
                ]
            })
    
    return {
        "mean_groundedness": np.mean(all_scores),
        "median_groundedness": np.median(all_scores),
        "min_groundedness": np.min(all_scores),
        "fraction_above_08": np.mean([s > 0.8 for s in all_scores]),
        "fraction_above_09": np.mean([s > 0.9 for s in all_scores]),
        "failure_cases": failure_cases,
        "failure_rate": len(failure_cases) / len(golden_queries)
    }
```

## Key Takeaways

1. **Groundedness is non-negotiable in banking**: Target > 0.85, never accept < 0.7
2. **Measure in real-time**: Check every production response
3. **Track trends**: Alert on downward trends
4. **Focus on numbers**: Numerical claims are highest risk
5. **Combine methods**: NLI + LLM judge + fact overlap gives best coverage
6. **Act on failures**: Block or flag responses below threshold
