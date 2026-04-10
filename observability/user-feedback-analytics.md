# User Feedback Analytics for Banking GenAI

## Why User Feedback Matters

Automated metrics measure what the system does. User feedback measures whether what the system does is useful. In banking GenAI, user feedback is the most direct signal of whether AI advice is actually helping customers.

## Feedback Collection Methods

### Thumbs Up/Down

The simplest and most common feedback mechanism:

```javascript
// Frontend: Capture thumbs feedback
function submitFeedback(responseId, rating, comment = null) {
  fetch('/api/v1/feedback', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      response_id: responseId,
      rating: rating,  // 'thumbs_up' or 'thumbs_down'
      comment: comment,  // Optional text feedback
      timestamp: new Date().toISOString(),
      page_url: window.location.pathname,
    })
  });
}
```

```python
# Backend: Record feedback
feedback_total = Counter(
    'user_feedback_total',
    'Total user feedback submissions',
    ['rating', 'product_line', 'model']
)

feedback_rate = Gauge(
    'user_satisfaction_rate',
    'User satisfaction rate (thumbs up / total)',
    ['product_line']
)

def record_feedback(response_id, rating, product_line, model, comment=None):
    feedback_total.labels(
        rating=rating, product_line=product_line, model=model
    ).inc()

    if comment:
        # Store comment for qualitative analysis
        store_feedback_comment(response_id, comment, rating)

    # Update satisfaction rate
    update_satisfaction_rate(product_line)
```

### Detailed Feedback Form

For more detailed feedback after important interactions (e.g., mortgage advice):

```python
FEEDBACK_CATEGORIES = {
    'accuracy': 'Was the information accurate?',
    'helpfulness': 'Was the advice helpful?',
    'clarity': 'Was the response clear and easy to understand?',
    'completeness': 'Did the response address all your questions?',
    'trustworthiness': 'Did you feel confident in this advice?',
}

FEEDBACK_SCALE = {
    1: 'Very Poor',
    2: 'Poor',
    3: 'Acceptable',
    4: 'Good',
    5: 'Excellent',
}

feedback_score = Histogram(
    'user_feedback_score',
    'User feedback scores by category',
    ['category', 'product_line'],
    buckets=[1, 2, 3, 4, 5]
)
```

### Implicit Feedback Signals

Users also signal quality through behavior:

```python
implicit_signals = {
    'conversation_abandonment': 'User stopped mid-conversation',
    'query_reformulation': 'User rephrased and asked again',
    'follow_up_question': 'User asked a clarifying follow-up',
    'response_copy': 'User copied the response',
    'session_duration': 'Time spent reading the response',
    'return_within_hour': 'User returned shortly after (dissatisfaction?)',
    'escalation_to_human': 'User requested human agent',
}

implicit_feedback = Counter(
    'implicit_feedback_total',
    'Implicit user feedback signals',
    ['signal_type', 'product_line', 'model']
)
```

## Satisfaction Metrics

### Net Promoter Score (NPS)

```python
def calculate_nps(feedback_scores):
    """
    Calculate Net Promoter Score.
    Promoters: 4-5, Passives: 3, Detractors: 1-2
    NPS = % Promoters - % Detractors
    """
    total = len(feedback_scores)
    promoters = sum(1 for s in feedback_scores if s >= 4)
    detractors = sum(1 for s in feedback_scores if s <= 2)

    return ((promoters - detractors) / total) * 100
```

### Customer Satisfaction Score (CSAT)

```python
def calculate_csat(feedback_scores):
    """
    Calculate CSAT as percentage of satisfied users (4-5 ratings).
    """
    satisfied = sum(1 for s in feedback_scores if s >= 4)
    return (satisfied / len(feedback_scores)) * 100
```

### Feedback Rate

```python
feedback_submission_rate = Gauge(
    'feedback_submission_rate',
    'Percentage of responses that received feedback',
    ['product_line']
)

# Industry benchmark: 5-15% of users provide feedback
# If rate drops significantly, the feedback mechanism may be broken
```

## Feedback Dashboard

```
┌──────────────────────────────────────────────────────────────┐
│  USER SATISFACTION - GenAI PLATFORM                           │
├──────────────┬──────────────┬──────────────┬─────────────────┤
│  CSAT        │  NPS         │  Feedback    │  Thumbs Up     │
│  Rate        │              │  Rate        │  Ratio         │
│              │              │              │                │
│  87%         │  +42         │  8.2%        │  82% / 18%     │
│  (target:    │  (target:    │  (target:    │  (target:      │
│   > 80%)     │   > +30)     │   > 5%)      │   > 75%)       │
└──────────────┴──────────────┴──────────────┴─────────────────┘

┌──────────────────────────────┬───────────────────────────────┐
│  Satisfaction Trend (weekly) │  Feedback by Product Line     │
│  95% ┤                     │                               │
│      │       ╱╲             │  Mortgage:     4.2/5.0 (89%)  │
│  85% ┤──────╱──╲──          │  Investment:   4.0/5.0 (85%)  │
│      │  ╱╲╱    ╲  ╱╲       │  Credit Card:  4.3/5.0 (91%)  │
│  75% ┤╱        ╲╱  ╲       │  General:      3.8/5.0 (80%)  │
│      └─────────────────     │                               │
│      W1  W2  W3  W4  W5     └───────────────────────────────┘
└──────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  Thumbs Down Reasons (from comments, auto-categorized)        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Outdated information       34%  [████████████]        │ │
│  │  Incorrect calculation      22%  [████████]            │ │
│  │  Too generic response       18%  [██████]              │ │
│  │  Missing details            14%  [█████]               │ │
│  │  Confusing explanation       8%  [███]                 │ │
│  │  Other                       4%  [█]                   │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  User Escalation to Human Agent                               │
│  Rate: 3.2% of all conversations (target: < 5%)              │
│  Trend: -0.5% vs last week (improving)                       │
│  Top reasons for escalation:                                  │
│    1. Complex scenario not handled (45%)                     │
│    2. Customer preference (30%)                              │
│    3. Dissatisfaction with AI response (25%)                 │
└──────────────────────────────────────────────────────────────┘
```

## Feedback-Driven Improvement

### Closing the Loop

```python
class FeedbackReviewQueue:
    """Queue negative feedback for human review."""

    def __init__(self):
        self.queue = []

    def add_for_review(self, feedback):
        """Add negative feedback to review queue."""
        if feedback.rating == 'thumbs_down':
            self.queue.append({
                'response_id': feedback.response_id,
                'product_line': feedback.product_line,
                'model': feedback.model,
                'comment': feedback.comment,
                'timestamp': feedback.timestamp,
                'priority': self._calculate_priority(feedback),
            })

    def _calculate_priority(self, feedback):
        """Higher priority for critical product lines and severe feedback."""
        priority = 1
        if feedback.product_line in ['mortgage', 'investment']:
            priority += 1
        if feedback.comment and any(
            word in feedback.comment.lower()
            for word in ['wrong', 'incorrect', 'misleading', 'complaint']
        ):
            priority += 2
        return priority
```

### A/B Testing with Feedback

```python
def analyze_ab_test_feedback(test_id):
    """Compare user feedback between A/B test variants."""
    variant_a_feedback = get_feedback_for_variant(test_id, 'A')
    variant_b_feedback = get_feedback_for_variant(test_id, 'B')

    return {
        'variant_a': {
            'csat': calculate_csat(variant_a_feedback),
            'thumbs_up_rate': thumbs_up_rate(variant_a_feedback),
            'escalation_rate': escalation_rate(variant_a_feedback),
            'sample_size': len(variant_a_feedback),
        },
        'variant_b': {
            'csat': calculate_csat(variant_b_feedback),
            'thumbs_up_rate': thumbs_up_rate(variant_b_feedback),
            'escalation_rate': escalation_rate(variant_b_feedback),
            'sample_size': len(variant_b_feedback),
        },
    }
```

## Feedback Correlation with Quality Metrics

```
┌─────────────────────────────────────────────────────────────┐
│  FEEDBACK vs AUTOMATED QUALITY CORRELATION                   │
│                                                             │
│  Automated Quality Score                                    │
│  1.0 ┤                                                      │
│      │     ●  ●  ●                                         │
│  0.8 ┤   ●    ●     ●    <- High correlation area          │
│      │  ●   ●  ●  ●                                        │
│  0.6 ┤ ●  ●    ●                                            │
│      │●   ●                                                  │
│  0.4 ┤ ●   <- Low scores, mixed feedback (hallucination?)   │
│      │                                                      │
│  0.2 ┤ ●                                                    │
│      │                                                      │
│  0.0 ┼────┬────┬────┬────┬────┬────┬────┬────               │
│      0.0  0.2  0.4  0.6  0.8  1.0                          │
│                    User Feedback Score                       │
│                                                             │
│  Outliers (high quality score, low feedback) indicate       │
│  responses that are technically correct but not helpful.    │
│  These are candidates for prompt/response style improvement.│
└─────────────────────────────────────────────────────────────┘
```

## Common Feedback Tracking Mistakes

1. **No feedback mechanism at all**: Building GenAI features without a way for users to report problems is a regulatory risk in banking.

2. **Feedback not linked to response**: If you cannot connect a thumbs-down to the specific response, model, and context, the feedback is not actionable.

3. **Ignoring implicit signals**: Users who abandon a conversation or request a human agent are signaling dissatisfaction even without clicking thumbs-down.

4. **Not acting on feedback**: Collecting feedback without a review process erodes user trust. Customers notice when nothing changes.

5. **Feedback fatigue**: Asking for feedback after every response leads to declining response rates. Be strategic about when to ask.

6. **Not segmenting feedback**: Aggregate feedback hides problems. A 4.2/5.0 overall score could hide a 2.8/5.0 for a specific product line or model.

7. **No trend tracking**: "Our CSAT is 87%" is good. "Our CSAT dropped from 92% to 87% after the model update" is actionable.
