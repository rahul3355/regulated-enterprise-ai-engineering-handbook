# Case Study: Model Quality Degradation Undetected for 3 Weeks

## Executive Summary

A fine-tuned customer intent classification model gradually degraded in quality over 3 weeks before detection. The degradation caused misrouted customer inquiries, delayed loan applications, and incorrect escalation of low-priority tickets. The root cause was a silent dependency update that changed text preprocessing behavior, combined with missing model quality monitoring.

**Severity:** SEV-2 (Service Degradation)
**Duration:** 21 days (undetected), 4 days (detection to remediation)
**Impact:** 18,400 misrouted inquiries, 340 delayed loan applications, 2,100 incorrectly escalated tickets
**Customer satisfaction impact:** NPS dropped from 72 to 58 during the period
**Financial impact:** $280,000 (overtime for backlogged work, customer compensation)

---

## Background and Context

### The System

"IntentRouter" is a fine-tuned BERT-based model that classifies incoming customer inquiries into 47 intent categories:
- Loan applications (mortgage, personal, business)
- Account services (balance, statements, disputes)
- Card services (activation, replacement, limits)
- Complaints and escalations
- General inquiries

The classification output determines:
- Which team receives the inquiry
- Priority level assignment
- Expected SLA timer
- Automated response templates

### Architecture

```
Customer Inquiry (text)
    |
    v
Text Preprocessing Pipeline
    - Lowercase
    - Tokenize (spaCy)
    - Remove stop words
    - Stem/lemmatize
    - Remove special characters
    |
    v
Intent Classification Model (fine-tuned BERT)
    |
    v
Routing Decision
    - Team assignment
    - Priority level
    - SLA timer
```

### Monitoring (Insufficient)

The team monitored:
- Model inference latency (p50, p95, p99)
- API error rates (5xx responses)
- Request volume
- **What was missing:** Classification distribution drift, accuracy metrics, human correction rate

---

## Timeline of Events

```mermaid
timeline
    title Model Degradation Timeline (3 Weeks Undetected)
    section Week -3 to Week 0
        Day -21 : spaCy library auto-updated<br/>from 3.7.2 to 3.7.4
        : New version changes<br/>lemmatization behavior<br/>for financial terms
        : No model quality<br/>alerts configured
        : No classification<br/>distribution monitoring
    section Week 1 (Days -21 to -14)
        Day -20 : "loan modification" now<br/>lemmatized differently,<br/>model confidence drops
        Day -18 : Misrouting rate increases<br/>from 2.1% to 4.8%
        : No alerts triggered<br/>(no monitoring exists)
        Day -16 : Support agents notice<br/>more misrouted tickets<br/>but attribute to volume
        Day -14 : Loan application misrouting<br/>begins: 12 apps sent to<br/>wrong team
    section Week 2 (Days -14 to -7)
        Day -12 : Misrouting rate at 8.3%<br/>NPS begins declining
        : Customer complaints<br/>about "wrong department"<br/>increase 40%
        Day -10 : 47 loan applications<br/>misrouted to card services<br/>team
        Day -8 : Backlog grows in<br/>loan processing team<br/>(not receiving correct apps)
        Day -7 : Weekly metrics reviewed:<br/>latency normal, error<br/>rates normal - no quality check
    section Week 3 (Days -7 to 0)
        Day -5 : Misrouting rate peaks<br/>at 12.1%
        : 340 loan applications<br/>delayed by 5-12 days
        Day -4 : Complaint escalations<br/>increase 200%
        Day -3 : Senior support manager<br/>requests metrics on<br/>misrouted inquiries
        Day -2 : Data analyst pulls<br/>classification distribution<br/>and notices dramatic shift
        Day -1 : Analyst escalates to<br/>ML engineering team
    section Detection and Remediation (Day 0 to Day 4)
        Day 0 09:00 : ML engineer reviews<br/>classification distribution,<br/>confirms significant drift
        Day 0 10:30 : Root cause investigation<br/>begins
        Day 0 14:00 : spaCy version change<br/>identified as trigger
        Day 0 15:00 : SEV-2 declared
        Day 0 16:00 : Rollback to spaCy 3.7.2<br/>deployed
        Day 1 : Model re-evaluated on<br/>test set: accuracy back<br/>to 96.2%
        Day 2 : Backlogged inquiries<br/>manually reclassified<br/>and re-routed
        Day 3 : Loan applications<br/>expedited, customers<br/>contacted with apologies
        Day 4 : Monitoring dashboards<br/>deployed with drift<br/>detection alerts
```

### The Silent Change

The spaCy 3.7.4 update changed lemmatization behavior for compound financial terms:

```python
# spaCy 3.7.2 (expected behavior)
nlp("loan modification")  # -> ["loan", "modification"]

# spaCy 3.7.4 (changed behavior)
nlp("loan modification")  # -> ["loan", "modify"]  # "modification" -> "modify"

# Impact on model input:
# Training data: "loan modification" -> class: loan_services
# Production after update: "loan modify" -> class: general_inquiry (WRONG)
```

The model was trained on preprocessed text where "modification" was preserved as-is. The new lemmatization changed the input distribution, causing the model to produce incorrect classifications.

---

## Root Cause Analysis

### Technical Root Causes

1. **Unpinned Dependency Version**
   - `requirements.txt` specified `spacy>=3.7.0` instead of `spacy==3.7.2`
   - CI/CD pipeline pulled the latest compatible version automatically
   - No dependency lock file (no `requirements.lock` or `poetry.lock`)

2. **Missing Model Quality Monitoring**
   - No monitoring of classification distribution over time
   - No tracking of model confidence scores
   - No comparison of model predictions vs. human corrections
   - No drift detection on input text distribution

3. **No Model Retraining Pipeline**
   - The model was trained once and deployed without automated retraining
   - No feedback loop from human corrections back into training data
   - No scheduled model evaluation against a held-out test set

4. **Insufficient Input Validation**
   - No validation that preprocessing output matches expected distributions
   - No alerting on changes to token frequency or vocabulary coverage

### Organizational Root Causes

1. **ML as a "Deploy and Forget" System**
   - The model was treated like traditional software: deploy and monitor for errors
   - ML-specific concerns (drift, accuracy decay, data quality) were not part of operations
   - No ML engineer on the on-call rotation

2. **Metric Myopia**
   - The team monitored technical metrics (latency, errors) but not business metrics (accuracy, misrouting rate)
   - Weekly reviews focused on system health, not model health
   - No one was responsible for checking classification quality

3. **Support Team Feedback Loop Broken**
   - Support agents noticed misrouting but had no direct channel to report it to ML engineering
   - Misrouted tickets were manually re-routed without tracking the pattern
   - No aggregate view of human corrections existed

4. **Dependency Management Process Gap**
   - No process for evaluating the impact of dependency updates on ML models
   - Dependency updates were treated as routine, not as potential model-affecting changes

---

## What Went Wrong Technically

### The Dependency Chain

```
requirements.txt: spacy>=3.7.0
        |
        v
CI/CD pulls spacy 3.7.4 (latest compatible)
        |
        v
New lemmatization behavior changes model input
        |
        v
Model produces different classifications
        |
        v
No monitoring detects the change
        |
        v
Misrouted inquiries accumulate for 21 days
```

### Missing Monitoring

```python
# What SHOULD have been monitored:

# 1. Classification distribution
def monitor_classification_distribution(predictions):
    """Alert if class distribution shifts beyond threshold."""
    baseline_distribution = get_baseline_distribution()
    current_distribution = compute_current_distribution(predictions)

    kl_divergence = compute_kl_divergence(baseline, current)
    if kl_divergence > THRESHOLD:
        alert(f"Classification distribution drift detected: KL={kl_divergence:.4f}")

# 2. Human correction rate
def monitor_human_corrections():
    """Track how often humans override model predictions."""
    correction_rate = corrections_24h / total_predictions_24h
    if correction_rate > baseline_rate * 2:
        alert(f"Human correction rate doubled: {correction_rate:.2%}")

# 3. Model confidence
def monitor_confidence_scores(confidence_scores):
    """Alert if model confidence drops significantly."""
    avg_confidence = np.mean(confidence_scores)
    if avg_confidence < baseline_confidence - 0.1:
        alert(f"Model confidence dropped: {avg_confidence:.3f}")

# NONE of these existed in production.
```

---

## What Went Wrong Organizationally

1. **No ML Operations Discipline**: The team had MLOps tooling but did not use it for production monitoring. Model monitoring was an afterthought.

2. **Siloed Teams**: Support operations and ML engineering operated in silos. Support agents had no mechanism to report systematic classification errors.

3. **Metric Dashboard Gaps**: The engineering dashboard showed system health but not model health. Leadership reviews focused on uptime, not accuracy.

4. **No Ownership**: No single person or team owned model quality in production. The ML team owned training, the ops team owned infrastructure, and nobody owned the gap between them.

---

## Immediate Response and Mitigation

### First 24 Hours

1. **Rollback**: spaCy was pinned to 3.7.2 and redeployed
2. **Verification**: Model accuracy verified against held-out test set: 96.2% (back to baseline)
3. **Impact Assessment**: Pulled all classifications from the 21-day period and compared with baseline distribution
4. **Backlog Identification**: Identified 340 loan applications and 2,100 tickets that were misrouted

### Days 1-4

1. **Backlog Reprocessing**: All misrouted inquiries were manually reclassified and re-routed
2. **Customer Outreach**: Customers with delayed loan applications were contacted with apologies and expedited processing
3. **Compensation**: Overtime costs for reprocessing teams totaled $280,000
4. **Monitoring Deployment**: Classification distribution monitoring and drift detection were deployed

---

## Long-Term Fixes and Systemic Changes

### Technical Fixes

1. **Dependency Pinning and Lock Files**
   ```
   # requirements.txt (AFTER)
   spacy==3.7.2  # Pinned version
   # Plus: requirements.lock generated by pip-compile
   ```

2. **Model Quality Monitoring Dashboard**
   - Real-time classification distribution with baseline comparison
   - KL divergence and PSI (Population Stability Index) tracking
   - Human correction rate monitoring
   - Model confidence score tracking
   - Input text distribution drift detection

3. **Automated Drift Alerting**
   ```python
   # Drift detection service
   class ModelDriftDetector:
       def __init__(self, baseline_distribution, kl_threshold=0.05):
           self.baseline = baseline_distribution
           self.threshold = kl_threshold

       def check(self, recent_predictions):
           current = self._compute_distribution(recent_predictions)
           kl = self._kl_divergence(self.baseline, current)
           if kl > self.threshold:
               self._alert(kl)
   ```

4. **Feedback Loop from Human Corrections**
   - Every human correction logged with: original prediction, corrected class, confidence score
   - Weekly retraining pipeline incorporating corrections
   - Monthly model evaluation report

5. **Dependency Update Process**
   - All dependency updates require ML model re-evaluation before deployment
   - Automated test: run model on validation set after dependency update, compare accuracy
   - No automatic dependency updates in production without manual approval

### Process Changes

1. **ML Quality as a SEV Metric**: Model accuracy degradation is now a SEV-triggering condition
2. **Weekly Model Health Review**: ML engineers review model quality metrics weekly
3. **Support-ML Feedback Channel**: Direct channel for support agents to report systematic issues
4. **Model Retraining Schedule**: Monthly retraining with human corrections incorporated
5. **Dependency Change Management**: Dependency updates affecting ML pipelines require ML team approval

### Cultural Changes

1. **ML as a Living System**: Models require ongoing monitoring and maintenance, not just initial training
2. **Human-in-the-Loop Value**: Human corrections are a valuable signal, not just operational overhead
3. **Data-Driven Operations**: Decisions about model health are based on metrics, not anecdotes

---

## Lessons Learned

1. **Dependencies Can Break ML Models Silently**: A library update that changes preprocessing behavior can degrade model accuracy without any code changes.

2. **Monitor Model Outputs, Not Just System Metrics**: Latency and error rates tell you nothing about model quality.

3. **Human Corrections Are a Canary**: When humans frequently override model predictions, the model is degrading.

4. **ML Requires Different Operations Discipline**: Traditional SRE practices are necessary but insufficient for ML systems.

5. **Feedback Loops Are Essential**: The gap between model predictions and ground truth must be continuously measured and fed back into training.

6. **Dependency Management Is Model Management**: For ML systems, dependency updates are model-affecting changes.

---

## Interview Questions Derived From This Case Study

1. **ML Operations**: "How would you monitor a production ML model for quality degradation? What metrics would you track?"

2. **System Design**: "Design a monitoring system for a text classification model in production. What signals would you use to detect degradation?"

3. **Incident Response**: "You discover that a model has been producing incorrect predictions for 3 weeks. How do you assess the impact and remediate?"

4. **Dependency Management**: "How do you manage dependency updates in an ML system? What testing is required?"

5. **Feedback Loops**: "How would you design a feedback loop where human corrections improve the model over time?"

6. **Drift Detection**: "What is data drift? How do you detect it? What actions do you take when drift is detected?"

7. **Organizational**: "Who should own model quality in production? How do you structure teams to ensure ongoing model health?"

---

## Cross-References

- See `../data-engineering/data-drift-monitoring.md` for drift detection techniques
- See `../mlops/model-monitoring.md` for ML production monitoring
- See `../incident-management/detection-and-alerting.md` for alert design principles
- See `../engineering-culture/ml-ops-discipline.md` for MLOps best practices
- See `../databases/vector-databases.md` for vector database operations
