# Model Risk Management -- Ongoing Monitoring, Validation, Lifecycle

## Overview

Model risk management (MRM) is the end-to-end process of managing risks associated with models throughout their lifecycle -- from development through retirement. While SR 11-7 (see `sr11-7-model-risk.md`) provides the regulatory framework, this document covers the practical engineering implementation of ongoing MRM.

**Key principle**: Model risk does not end at deployment. Models degrade, data shifts, and business contexts change. Ongoing monitoring, validation, and governance are essential.

## Model Lifecycle

```
MODEL LIFECYCLE
═══════════════

  ┌─────────┐    ┌───────────┐    ┌───────────┐    ┌───────────┐
  │  DESIGN  ├───►│ DEVELOP   ├───►│  VALIDATE  ├───►│  DEPLOY   │
  └─────────┘    └───────────┘    └───────────┘    └───────────┘
                                                        │
  ┌─────────┐    ┌───────────┐    ┌───────────┐         │
  │ RETIRE  │◄───┤ REMEDIATE  │◄───┤  MONITOR   │◄────────┘
  └─────────┘    └───────────┘    └───────────┘
```

### Phase 1: Design

```
DESIGN PHASE CHECKLIST:
├── Business problem clearly defined
├── Model purpose documented
├── Intended users identified
├── Success criteria defined
├── Risk classification assigned (see sr11-7-model-risk.md)
├── Data requirements documented
├── Methodology selected and justified
├── Alternatives considered
├── Constraints identified (technical, regulatory, ethical)
├── Stakeholders identified
└── Design document approved
```

### Phase 2: Development

```
DEVELOPMENT PHASE CHECKLIST:
├── Data sourced and documented
├── Data quality assessed
├── Training/validation/test sets defined
├── Feature engineering documented
├── Model trained and tuned
├── Performance evaluated on holdout data
├── Bias and fairness assessed
├── Robustness tested
├── Stress scenarios evaluated
├── Code reviewed
├── Unit tests passed
├── Integration tests passed
├── Documentation complete (model card)
└── Ready for validation
```

### Phase 3: Validation

```
VALIDATION PHASE CHECKLIST:
├── Independent validator assigned (not development team)
├── Conceptual soundness assessed
├── Implementation accuracy verified
├── Outcome analysis completed
├── Bias and fairness independently verified
├── Stress test results reviewed
├── Limitations documented and communicated
├── Validation report produced
├── Findings remediated
└── Validation approved
```

### Phase 4: Deployment

```
DEPLOYMENT PHASE CHECKLIST:
├── Deployment approval obtained
├── Production environment prepared
├── Monitoring implemented
├── Alerting configured
├── Runbook created
├── On-call rotation set up
├── Incident response procedures tested
├── Rollback procedures tested
├── User training completed
├── Limitations communicated to users
└── Go-live approved
```

### Phase 5: Ongoing Monitoring

See "Ongoing Monitoring" section below.

### Phase 6: Remediation or Retirement

```
REMEDIATION TRIGGERS:
├── Performance degradation beyond tolerance
├── Significant input data distribution shift
├── Regulatory requirement change
├── Business process change
├── Model incident requiring investigation
├── Validation finding requiring remediation
└── User feedback indicating issues

RETIREMENT TRIGGERS:
├── Model no longer fit for purpose
├── Replaced by newer model
├── Business need no longer exists
├── Regulatory prohibition
├── Cost of remediation exceeds benefit
└── Model risk exceeds appetite

RETIREMENT CHECKLIST:
├── Retirement decision documented
├── Replacement plan (if applicable)
├── User notification
├── Downstream system notification
├── Data disposition (retain, archive, delete)
├── Knowledge retention (documentation, lessons learned)
├── System decommissioning
├── Post-retirement review
└── Model inventory updated
```

## Ongoing Monitoring

### Performance Monitoring

```
PERFORMANCE METRICS BY MODEL TYPE:
├── Classification models:
│   ├── Accuracy, precision, recall, F1
│   ├── AUC-ROC, Gini coefficient
│   ├── Confusion matrix (by segment)
│   ├── Calibration (Brier score, reliability diagram)
│   └── Threshold analysis (optimal threshold drift)
├── Regression models:
│   ├── MAE, RMSE, R-squared
│   ├── Residual analysis (distribution, patterns)
│   ├── Prediction intervals (coverage)
│   └── Segment-level performance
├── Ranking models:
│   ├── NDCG, MAP
│   ├── Hit rate@K
│   └── Rank correlation
├── LLM/GenAI models:
│   ├── Human evaluation scores
│   ├── Task-specific accuracy
│   ├── Hallucination rate
│   ├── Safety violation rate
│   ├── Prompt injection success rate
│   ├── Response latency
│   └── User satisfaction scores
└── All models:
    ├── Volume of predictions
    ├── Override rate (human overrides of model output)
    ├── User feedback scores
    └── Incident count
```

### Stability Monitoring

```
STABILITY METRICS:
├── Population Stability Index (PSI):
│   ├── PSI < 0.1: No significant change
│   ├── PSI 0.1-0.25: Moderate change, investigate
│   └── PSI > 0.25: Significant change, action required
├── Characteristic Stability Index (CSI):
│   ├── Per-feature stability
│   └── Identifies which features are drifting
├── Feature distribution monitoring:
│   ├── Mean, variance, skewness, kurtosis
│   ├── Missing rate
│   ├── Out-of-range rate
│   └── Novel value rate (for categorical features)
├── Concept drift detection:
│   ├── Relationship between features and target changing
│   ├── Detected via performance degradation
│   └── Methods: ADWIN, DDM, EDDM, Page-Hinkley
└── Data drift detection:
    ├── Input data distribution changing
    ├── Detected via PSI, KS test, MMD
    └── Alert before performance degradation
```

```python
class ModelStabilityMonitor:
    """Monitor model stability for drift and degradation."""

    def check_stability(self, model_id: str, period: str) -> StabilityReport:
        model = self.get_model(model_id)
        current_data = self.get_input_data(model_id, period)
        baseline_data = model.training_data

        return StabilityReport(
            model_id=model_id,
            period=period,
            psi=self.compute_psi(current_data, baseline_data),
            csi=self.compute_csi(current_data, baseline_data),
            feature_drift=self.detect_feature_drift(current_data, baseline_data),
            concept_drift=self.detect_concept_drift(current_data, baseline_data),
            recommendations=self.generate_recommendations(
                model_id, current_data, baseline_data
            ),
        )

    def compute_psi(self, current: DataFrame, baseline: DataFrame) -> Dict[str, float]:
        """Compute Population Stability Index for each feature."""
        psi_scores = {}
        for column in current.columns:
            if current[column].dtype == "object":
                psi_scores[column] = self.psi_categorical(
                    current[column], baseline[column]
                )
            else:
                psi_scores[column] = self.psi_continuous(
                    current[column], baseline[column]
                )
        return psi_scores

    def generate_recommendations(
        self, model_id: str, current: DataFrame, baseline: DataFrame
    ) -> List[str]:
        """Generate actionable recommendations based on stability analysis."""
        recommendations = []
        psi = self.compute_psi(current, baseline)

        max_psi = max(psi.values())
        if max_psi > 0.25:
            recommendations.append(
                f"Significant drift detected (max PSI: {max_psi:.2f}). "
                f"Model recalibration recommended."
            )
        elif max_psi > 0.1:
            recommendations.append(
                f"Moderate drift detected (max PSI: {max_psi:.2f}). "
                f"Investigate root cause."
            )

        # Check specific features
        for feature, psi_value in psi.items():
            if psi_value > 0.25:
                recommendations.append(
                    f"Feature '{feature}' has drifted significantly (PSI: {psi_value:.2f})."
                )

        return recommendations
```

### Input Data Quality Monitoring

```
DATA QUALITY METRICS:
├── Completeness: % of non-null values
├── Validity: % of values within expected range
├── Consistency: % of values consistent with related fields
├── Timeliness: Data freshness
├── Uniqueness: Duplicate rate
└── Accuracy: Error rate (compared to source of truth)
```

### Usage Monitoring

```
USAGE METRICS:
├── Prediction volume (over time, by segment)
├── User access patterns
├── Query patterns (input distributions)
├── Response latency (p50, p95, p99)
├── Error rate (API errors, model errors)
├── Timeout rate
└── Cost per prediction
```

## Outcome Analysis

### Backtesting

Compare model predictions to actual outcomes to assess calibration and accuracy.

```python
class ModelBacktesting:
    """Backtest model predictions against actual outcomes."""

    def backtest(self, model_id: str, period: str) -> BacktestReport:
        predictions = self.get_predictions(model_id, period)
        actuals = self.get_actual_outcomes(model_id, period)

        return BacktestReport(
            model_id=model_id,
            period=period,
            accuracy_metrics=self.compute_accuracy(predictions, actuals),
            calibration=self.assess_calibration(predictions, actuals),
            segment_analysis=self.analyze_by_segment(predictions, actuals),
            benchmark_comparison=self.compare_to_benchmark(predictions, actuals),
            stability_analysis=self.assess_stability(predictions, actuals),
            conclusions=self.draw_conclusions(predictions, actuals),
        )

    def compute_accuracy(
        self, predictions: List[float], actuals: List[float]
    ) -> AccuracyMetrics:
        return AccuracyMetrics(
            accuracy=accuracy_score(actuals, [p > 0.5 for p in predictions]),
            precision=precision_score(actuals, [p > 0.5 for p in predictions]),
            recall=recall_score(actuals, [p > 0.5 for p in predictions]),
            f1=f1_score(actuals, [p > 0.5 for p in predictions]),
            auc=roc_auc_score(actuals, predictions),
        )
```

### Benchmarking

Compare model performance to alternative approaches:

```
BENCHMARKING APPROACHES:
├── Compare to previous model version
├── Compare to simpler baseline (e.g., logistic regression)
├── Compare to human performance
├── Compare to random baseline
├── Compare to industry benchmarks
└── Compare to business rules
```

### Expert Review

Have subject matter experts review model outputs for reasonableness:

```
EXPERT REVIEW PROCESS:
├── Sample model outputs (stratified by segment and score)
├── Expert reviews for reasonableness
├── Document expert feedback
├── Identify patterns of disagreement
├── Escalate significant issues
└── Incorporate feedback into model improvement
```

## Model Risk Reporting

### Monthly Model Performance Report

```
MONTHLY MODEL REPORT:
├── Executive summary:
│   ├── Overall model health (green/amber/red)
│   ├── Key changes since last report
│   └── Actions required
├── Performance metrics:
│   ├── Current vs. previous period
│   ├── Current vs. baseline (development)
│   └── Trends over time
├── Stability metrics:
│   ├── PSI and CSI values
│   └── Drift detection results
├── Data quality:
│   ├── Input data quality scores
│   └── Data issues identified
├── Usage statistics:
│   ├── Volume trends
│   ├── Latency metrics
│   └── Error rates
├── Incidents:
│   ├── Summary of model-related incidents
│   └── Status of remediation actions
├── Validation status:
│   ├── Last validation date
│   └── Next validation due
└── Recommendations:
    ├── Required actions
    └── Suggested improvements
```

### Quarterly Model Risk Report to Management

```
QUARTERLY MANAGEMENT REPORT:
├── Model inventory summary
├── Risk classification summary
├── Validation status summary
├── Performance trends
├── Incident summary
├── Remediation tracking
├── Resource requirements
├── Risk appetite assessment
└── Strategic recommendations
```

## Model Documentation

### Model Card

```
MODEL CARD TEMPLATE:
├── Model details:
│   ├── Name, version, ID
│   ├── Owner, developer
│   ├── Date created, last updated
│   └── Risk classification
├── Intended use:
│   ├── Purpose
│   ├── Intended users
│   └── Out-of-scope uses
├── Model architecture:
│   ├── Algorithm type
│   ├── Parameters
│   ├── Training methodology
│   └── Hyperparameters
├── Training data:
│   ├── Sources
│   ├── Size, timeframe
│   ├── Preprocessing steps
│   └── Known limitations
├── Evaluation data:
│   ├── Test set description
│   ├── Performance metrics
│   └── Segment-level performance
├── Ethical considerations:
│   ├── Bias assessment
│   ├── Fairness analysis
│   └── Mitigation measures
├── Limitations:
│   ├── Known failure modes
│   ├── Edge cases
│   └── Conditions where model should not be used
├── Deployment:
│   ├── Environment
│   ├── Dependencies
│   └── Monitoring setup
└── Version history:
    ├── Changes per version
    └── Validation status per version
```

## Common Interview Questions

### Question 1: "How do you detect model drift in production?"

**Good answer structure**:
1. Monitor input data distribution using PSI and feature-level statistics
2. Monitor model performance using actual outcomes (backtesting)
3. Monitor model confidence scores for shifts
4. Monitor business metrics that the model influences
5. Set alerting thresholds (PSI > 0.25 triggers investigation, > 0.1 triggers monitoring)
6. Use automated drift detection methods (ADWIN, KS tests)
7. Conduct periodic manual reviews with subject matter experts

### Question 2: "A model's performance has degraded. What do you do?"

**Good answer structure**:
1. Classify the severity: How much degradation? Is it within tolerance?
2. Investigate root cause: Data drift? Concept drift? Model decay? External event?
3. Assess impact: Which decisions are affected? Are consumers being harmed?
4. Implement short-term fix: Adjust thresholds, add human review, reduce automation
5. Develop long-term fix: Retrain model, update features, change methodology
6. Communicate: Notify stakeholders, document the issue, update risk register
7. Validate: Test the fix before deploying, follow change management process
8. Learn: Post-incident review, update monitoring, prevent recurrence

### Question 3: "How often should models be re-validated?"

**Good answer**:
Per SR 11-7, at least annually for high-risk models. But also: whenever there is a material change to the model, input data, or intended use; when monitoring detects significant degradation; after a model-related incident; when regulatory requirements change; or when the business context changes significantly. In practice, I recommend continuous monitoring with formal re-validation at least annually for critical/high-risk models, and trigger-based re-validation for any material change.
