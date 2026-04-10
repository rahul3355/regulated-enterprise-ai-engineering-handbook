# Hallucination Monitoring for Banking GenAI

## What Is Hallucination?

In GenAI, hallucination refers to the model generating confident-sounding but incorrect, fabricated, or unsupported information. In a banking context, hallucination is not just embarrassing -- it is a regulatory and legal risk.

**Banking hallucination examples**:
- Quoting an incorrect mortgage interest rate
- Stating a customer has a balance they do not have
- Inventing a bank policy that does not exist
- Providing investment advice based on outdated data
- Claiming a fee does not exist when it does

## Hallucination Severity Levels

```python
HALLUCINATION_SEVERITY = {
    'CRITICAL': {
        'description': 'Incorrect financial advice that could harm the customer',
        'examples': [
            'Wrong interest rate quoted',
            'Incorrect loan eligibility stated',
            'False information about fees or charges',
            'Incorrect regulatory compliance claimed',
        ],
        'response_time': 'Immediate page to on-call and ML team',
        'action': 'Disable the affected model/response path until fixed',
    },
    'WARNING': {
        'description': 'Inaccurate information that is caught by verification',
        'examples': [
            'Outdated product information referenced',
            'Minor factual inaccuracy in explanation',
            'Source document misattributed',
        ],
        'response_time': 'Ticket within business hours',
        'action': 'Flag for review, update knowledge base',
    },
    'INFO': {
        'description': 'Cosmetic issue or very low-risk inaccuracy',
        'examples': [
            'Slightly verbose explanation',
            'Minor formatting issue',
            'Non-critical date off by one day',
        ],
        'response_time': 'Added to quality improvement backlog',
        'action': 'Track for trend analysis',
    },
}
```

## Detection Methods

### 1. Source Grounding Check

The most reliable method: verify every claim in the response against the retrieved source documents.

```python
def check_source_grounding(response, source_documents) -> dict:
    """Check if all claims in the response are supported by source documents."""
    claims = extract_claims(response.content)
    results = {
        'total_claims': len(claims),
        'supported_claims': 0,
        'unsupported_claims': [],
        'grounding_score': 0.0,
    }

    for claim in claims:
        is_supported = verify_claim_against_sources(claim, source_documents)
        if is_supported:
            results['supported_claims'] += 1
        else:
            results['unsupported_claims'].append({
                'claim': claim.text,
                'claim_type': classify_claim_type(claim),
            })

    if results['total_claims'] > 0:
        results['grounding_score'] = (
            results['supported_claims'] / results['total_claims']
        )

    return results

def verify_claim_against_sources(claim, sources) -> bool:
    """Use NLI (Natural Language Inference) to verify a claim."""
    # Use an NLI model or LLM to determine if sources entail the claim
    for source in sources:
        entailment_score = compute_entailment(claim.text, source.content)
        if entailment_score > 0.85:
            return True
    return False
```

### 2. Contradiction Detection

```python
def detect_contradictions(response, source_documents) -> list:
    """Detect if response contradicts source documents."""
    contradictions = []

    response_facts = extract_facts(response.content)

    for fact in response_facts:
        for source in source_documents:
            contradiction_score = compute_contradiction(fact, source.content)
            if contradiction_score > 0.8:
                contradictions.append({
                    'response_claim': fact.text,
                    'source_text': source.snippet,
                    'contradiction_score': contradiction_score,
                })

    return contradictions
```

### 3. Numerical Consistency

Particularly important for banking -- verify numbers in the response:

```python
def check_numerical_consistency(response, source_documents) -> list:
    """Verify numerical claims against sources."""
    issues = []

    response_numbers = extract_numbers(response.content)
    source_numbers = extract_all_numbers([doc.content for doc in source_documents])

    for resp_num in response_numbers:
        # Check if this number appears in sources
        found = any(
            abs(resp_num.value - src_num.value) / max(resp_num.value, 1) < 0.01
            for src_num in source_numbers
            if resp_num.unit == src_num.unit
        )
        if not found and resp_num.is_claim:
            issues.append({
                'number': resp_num.value,
                'unit': resp_num.unit,
                'context': resp_num.context,
            })

    return issues
```

### 4. LLM-as-Judge Evaluation

Use a stronger model to evaluate a weaker model's output:

```python
async def evaluate_with_llm_judge(query, response, sources) -> dict:
    """Use GPT-4 to evaluate response quality of a cheaper model."""
    evaluation_prompt = f"""
    You are evaluating the accuracy of a financial advisory response.

    User Query: {query}
    AI Response: {response.content}
    Source Documents: {[doc.summary for doc in sources]}

    Evaluate:
    1. Does the response directly answer the query? (yes/no/partial)
    2. Are all factual claims supported by the source documents? (yes/no/partial)
    3. Does the response contain any fabricated information? (yes/no)
    4. Is the response free of contradictions with the sources? (yes/no)
    5. Overall accuracy score (1-5):

    Respond in JSON format with your evaluation.
    """

    evaluation = await call_llm(
        model="gpt-4-turbo",
        prompt=evaluation_prompt,
        temperature=0,
    )

    return parse_evaluation(evaluation)
```

## Hallucination Rate Tracking

```python
hallucination_detected = Counter(
    'hallucination_detected_total',
    'Total hallucinations detected',
    ['model', 'product_line', 'severity', 'detection_method']
)

grounding_score_dist = Histogram(
    'response_grounding_score',
    'Distribution of response grounding scores',
    ['model', 'product_line'],
    buckets=[i/10 for i in range(11)]
)

def record_hallucination(
    model: str,
    product_line: str,
    severity: str,
    detection_method: str,
    grounding_score: float = None
):
    """Record a detected hallucination."""
    hallucination_detected.labels(
        model=model, product_line=product_line,
        severity=severity, detection_method=detection_method
    ).inc()

    if grounding_score is not None:
        grounding_score_dist.labels(
            model=model, product_line=product_line
        ).observe(grounding_score)
```

## Hallucination Spike Detection

```yaml
- alert: HallucinationRateSpike
  expr: |
    rate(hallucination_detected_total{severity="critical"}[15m])
    > 0
  for: 2m
  labels:
    severity: critical
    team: ml-platform
  annotations:
    summary: "Critical hallucination detected"
    description: "A critical hallucination was detected for model {{ $labels.model }} in {{ $labels.product_line }}"

- alert: HighHallucinationRate
  expr: |
    sum(rate(hallucination_detected_total[1h]))
    /
    sum(rate(llm_calls_total[1h]))
    > 0.05
  for: 10m
  labels:
    severity: warning
    team: ml-platform
  annotations:
    summary: "Hallucination rate above 5%"
    description: "Current hallucination rate: {{ $value | humanizePercentage }}"

- alert: GroundingScoreDegradation
  expr: |
    histogram_quantile(0.5, rate(response_grounding_score[1h]))
    < 0.80
  for: 30m
  labels:
    severity: warning
    team: ml-platform
  annotations:
    summary: "Median grounding score below 0.80"
```

## Hallucination Dashboard

```
┌──────────────────────────────────────────────────────────────┐
│  HALLUCINATION MONITORING                                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Current Status:                                             │
│  Hallucination Rate: 2.1% (target: < 5%)  [OK]              │
│  Critical Rate: 0.02% (target: < 0.1%)  [OK]                │
│  Grounding Score: 0.91 (target: > 0.85)  [OK]               │
│                                                              │
│  Hallucinations by Severity (last 7 days):                   │
│  CRITICAL:  3   [  ■]                                        │
│  WARNING:  47   [    ██████]                                │
│  INFO:    156   [          ██████████████████]              │
│                                                              │
│  Hallucination Rate Trend (daily):                           │
│  8% ┤                                                       │
│     │              ╱╲                                       │
│  5% ┤─────────────╱──╲── SLO line                           │
│     │    ╱╲     ╱    ╲  ╱╲                                 │
│  2% ┤───╱──╲───╱──────╲╱──╲────                             │
│     └───────────────────────────────────                     │
│     Mon   Tue   Wed   Thu   Fri   Sat   Sun                  │
│                                                              │
│  Top Hallucination Categories:                               │
│  1. Outdated interest rate info    34%                       │
│  2. Misattributed policy source    22%                       │
│  3. Fabricated fee information     15%                       │
│  4. Incorrect eligibility criteria 12%                       │
│  5. Other                          17%                       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## Banking Hallucination Response Protocol

When a CRITICAL hallucination is detected:

```
1. IMMEDIATE (automated):
   - Flag the response in the audit log
   - If user is still in session, show correction message
   - Create incident ticket

2. WITHIN 15 MINUTES:
   - On-call engineer reviews the hallucination
   - Determines if model should be temporarily disabled
   - Notifies compliance team if customer-facing

3. WITHIN 1 HOUR:
   - Root cause analysis started
   - If systemic (not isolated), disable affected model path
   - Notify affected customers if necessary

4. WITHIN 24 HOURS:
   - Full root cause analysis completed
   - Fix implemented or mitigation deployed
   - Compliance report filed if required by regulation

5. WITHIN 1 WEEK:
   - Postmortem completed
   - Prevention measures implemented
   - Knowledge base updated to prevent recurrence
```

## Common Hallucination Monitoring Mistakes

1. **Only sampling for evaluation**: If you evaluate only 1% of responses, you miss 99% of hallucinations. Use automated checks on 100% of responses.

2. **No numerical verification**: Banking responses are full of numbers (rates, balances, fees). Verify every one.

3. **Relying solely on LLM-as-judge**: Using GPT-4 to evaluate GPT-3.5 outputs works, but GPT-4 can also hallucinate. Combine with source grounding checks.

4. **Not tracking by product line**: Hallucination rates vary by domain. Mortgage advice may have higher hallucination rates than general chat due to complexity.

5. **No trend analysis**: A jump from 2% to 5% hallucination rate is a serious regression. Alert on rate changes, not just absolute values.

6. **Ignoring false positives**: Over-aggressive hallucination detection creates noise. Track false positive rate and tune detection thresholds.
