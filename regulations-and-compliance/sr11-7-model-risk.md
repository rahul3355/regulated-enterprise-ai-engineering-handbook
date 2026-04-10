# SR 11-7 -- Model Risk Management (Federal Reserve/OCC)

## What Is SR 11-7 and Why It Matters

SR 11-7 ("Guidance on Model Risk Management") was issued jointly by the Federal Reserve and the OCC in 2011. It is the primary US regulatory guidance for model risk management at banking organizations.

**Key insight**: SR 11-7 applies to ALL models used by banking organizations -- not just traditional statistical models. Large Language Models (LLMs), generative AI systems, and any AI/ML model used in banking fall under this guidance.

**Who must comply**: All banking organizations supervised by the Federal Reserve and OCC, including bank holding companies, state member banks, and federal and state branches of foreign banks.

**Consequences of non-compliance**:
- Matters Requiring Attention (MRAs)
- Consent orders
- Cease and desist orders
- Civil money penalties
- Restrictions on business activities

## What Is a Model Under SR 11-7?

**Definition**: A model is a quantitative method, system, or approach that applies statistical, economic, financial, or mathematical theories, techniques, and assumptions to process input data into quantitative estimates.

**Key components**:
1. **Input data** -- The information feeding into the model
2. **Processing/analytics** -- How the model transforms inputs into outputs
3. **Output** -- The estimates or decisions the model produces

**What IS a model**:
- Credit scoring models
- Market risk models (VaR)
- Stress testing models
- Anti-money laundering (AML) models
- Fraud detection models
- Customer segmentation models
- Pricing models
- **LLMs used for credit decisions**
- **AI chatbots providing financial advice**
- **GenAI systems generating risk analysis**
- **Any AI system producing quantitative estimates**

**What is NOT a model**:
- Simple calculators
- Accounting systems (pure data processing)
- Systems with no analytical component

### LLM Classification Framework

```
IS YOUR LLM A MODEL UNDER SR 11-7?
                                   
Does it produce quantitative estimates or decisions?
├── YES ──> It IS a model
│   └── Proceed with full model risk management
│
└── NO ──> Does it support or influence model-related decisions?
    ├── YES ──> It IS a model (supporting model)
    │   └── Proceed with model risk management
    │
    └── NO ──> Is it used in a business context with model risk?
        ├── YES ──> May be subject to model risk management
        │   └── Consult Model Risk Management team
        │
        └── NO ──> Likely not a model, but document rationale
```

### Specific LLM Use Cases and Model Classification

| Use Case | Model? | Risk Tier | SR 11-7 Requirements |
|----------|--------|-----------|----------------------|
| Credit scoring using LLM | YES | High | Full validation, ongoing monitoring |
| AML alert triage | YES | High | Full validation, outcome analysis |
| Customer service chatbot (general info) | MAYBE | Low-Medium | Basic documentation, monitoring |
| Customer service chatbot (financial advice) | YES | High | Full validation, compliance review |
| Internal code generation tool | NO | Low | Basic IT controls |
| Fraud detection using LLM | YES | High | Full validation, backtesting |
| Trading signal generation | YES | Critical | Full validation, real-time monitoring |
| Document summarization (public docs) | MAYBE | Low | Basic documentation |
| Regulatory report drafting | YES | High | Human review, validation, audit trail |
| Employee training content generation | NO | Low | Basic content review |

## The Three Lines of Defense

SR 11-7 establishes a three lines of defense model:

### First Line: Model Development and Implementation

**Who**: Model developers, business users, IT implementers

**Responsibilities**:
- Develop models according to standards
- Test models before deployment
- Implement models correctly in production
- Monitor model performance
- Maintain model documentation

**Engineering requirements**:
```
FIRST LINE RESPONSIBILITIES:
├── Model development following standards
├── Unit testing of model code
├── Integration testing of model in production environment
├── Performance monitoring implementation
├── Model documentation (model card)
├── Input data validation
├── Output reasonableness checks
└── Incident reporting for model issues
```

### Second Line: Model Risk Management

**Who**: Independent model validation team, model risk governance

**Responsibilities**:
- Independent validation of models
- Model risk governance and oversight
- Model inventory management
- Challenge model development and performance
- Set model risk standards

**Engineering interaction**:
- Submit models for independent validation before production use
- Provide full access to model code, data, and documentation
- Respond to validation findings
- Implement required remediation actions

### Third Line: Internal Audit

**Who**: Internal audit function

**Responsibilities**:
- Independent review of model risk management
- Assess adequacy of first and second line controls
- Report to board and senior management

## Model Development Standards

### Model Design

**Requirements**:
- Clearly state the model's purpose and intended use
- Document all assumptions and limitations
- Select appropriate methodology
- Ensure model is conceptually sound
- Test model on relevant data samples

**Documentation**:
```
MODEL DESIGN DOCUMENT:
├── Model purpose and intended use
├── Model methodology
├── Key assumptions and limitations
├── Input data requirements and sources
├── Output description and interpretation
├── Model governance (owner, users, reviewers)
├── Development team and timeline
├── References to similar models
└── Known issues and constraints
```

### Input Data

**Requirements**:
- Identify and document all input data
- Assess input data quality
- Validate input data before processing
- Monitor input data for changes in distribution
- Document data lineage

**Engineering implementation**:
```python
class ModelInputValidation:
    def validate_inputs(self, inputs: ModelInput) -> ValidationResult:
        """Validate all input data before model processing."""
        results = []

        # Schema validation
        results.append(self.validate_schema(inputs))

        # Data quality checks
        results.append(self.validate_quality(inputs))

        # Range checks
        results.append(self.validate_ranges(inputs))

        # Distribution checks (compare to training distribution)
        results.append(self.validate_distribution(inputs))

        # Missing data analysis
        results.append(self.validate_completeness(inputs))

        return ValidationResult(
            passed=all(r.passed for r in results),
            details=results,
            timestamp=datetime.utcnow(),
        )

    def validate_distribution(self, inputs: ModelInput) -> DistributionCheck:
        """Check if input distribution matches training distribution."""
        current_stats = self.compute_statistics(inputs)
        training_stats = self.get_training_statistics()

        drift_metrics = self.compute_drift(
            current_stats, training_stats, method="psi"  # Population Stability Index
        )

        if drift_metrics.psi > 0.25:
            return DistributionCheck(
                passed=False,
                reason="Significant population drift detected",
                psi=drift_metrics.psi,
                recommendation="Model recalibration required",
            )
        return DistributionCheck(passed=True, psi=drift_metrics.psi)
```

### Model Testing

**Requirements**:
- Test model on data not used in development
- Assess model performance across relevant subsegments
- Test model under stress scenarios
- Compare model to alternatives
- Document testing results

**Testing methodology for LLMs**:
```
LLM MODEL TESTING:
├── Functional testing:
│   ├── Does the model produce expected outputs for known inputs?
│   ├── Does the model handle edge cases appropriately?
│   └── Does the model follow instructions consistently?
├── Performance testing:
│   ├── Accuracy against labeled test set
│   ├── Precision, recall, F1 (for classification)
│   ├── Calibration (do confidence scores match actual accuracy?)
│   └── Latency and throughput
├── Fairness testing:
│   ├── Performance across demographic groups
│   ├── Disparate impact analysis
│   └── Equal opportunity assessment
├── Robustness testing:
│   ├── Adversarial prompt testing
│   ├── Edge case handling
│   ├── Out-of-distribution behavior
│   └── Stress scenario performance
├── Security testing:
│   ├── Prompt injection resistance
│   ├── Data exfiltration resistance
│   └── Jailbreak resistance
└── Comparative testing:
    ├── Compare to existing model (if replacing)
    ├── Compare to human performance
    └── Compare to simpler baseline models
```

## Independent Model Validation

Before a model is used for any business purpose, it must be independently validated.

### Validation Scope

| Model Risk Tier | Validation Scope | Frequency |
|----------------|-----------------|-----------|
| **Critical** | Full validation + stress testing + alternative model comparison | Before deployment + annual re-validation |
| **High** | Full validation + stress testing | Before deployment + annual re-validation |
| **Medium** | Targeted validation + key assumption review | Before deployment + biennial re-validation |
| **Low** | Limited validation + documentation review | Before deployment + triennial re-validation |

### Validation Activities

```
INDEPENDENT VALIDATION CHECKLIST:
├── Conceptual soundness:
│   ├── Is the methodology appropriate for the purpose?
│   ├── Are assumptions reasonable and documented?
│   └── Are limitations understood?
├── Data quality:
│   ├── Is training data appropriate and well-documented?
│   ├── Are input data validated?
│   └── Is data lineage established?
├── Implementation:
│   ├── Is the model implemented correctly?
│   ├── Does code match the documented methodology?
│   └── Are there implementation errors?
├── Outcomes:
│   ├── Does the model perform as expected on test data?
│   ├── Is performance consistent with development results?
│   └── Are errors within acceptable bounds?
└── Use and limitations:
    ├── Is the model used for its intended purpose?
    ├── Are limitations communicated to users?
    └── Are there compensating controls for limitations?
```

## Ongoing Monitoring

Models must be monitored continuously after deployment.

### Monitoring Requirements

```
MODEL MONITORING PROGRAM:
├── Performance metrics:
│   ├── Accuracy, precision, recall (classification)
│   ├── MSE, MAE, R-squared (regression)
│   ├── AUC, Gini (discrimination)
│   └── Calibration (probability accuracy)
├── Stability metrics:
│   ├── Population Stability Index (PSI)
│   ├── Characteristic Stability Index (CSI)
│   ├── Feature distribution monitoring
│   └── Concept drift detection
├── Outcome analysis:
│   ├── Backtesting against actual outcomes
│   ├── Benchmarking against alternative models
│   └── Expert review of model outputs
├── Input data monitoring:
│   ├── Missing data rates
│   ├── Out-of-range values
│   ├── Distribution shifts
│   └── Data quality issues
├── Usage monitoring:
│   ├── Volume of predictions
│   ├── User access patterns
│   ├── Override rates and reasons
│   └── User feedback
└── Alerting thresholds:
    ├── Performance degradation > X%
    ├── PSI > 0.25
    ├── Error rate > X%
    ├── Input data quality issues
    └── Usage anomalies
```

```python
class ModelMonitoring:
    """Continuous model monitoring for SR 11-7 compliance."""

    def monitor_model(self, model_id: str, period: str) -> MonitoringReport:
        """Generate monitoring report for a model."""
        return MonitoringReport(
            model_id=model_id,
            period=period,
            performance=self.assess_performance(model_id, period),
            stability=self.assess_stability(model_id, period),
            data_quality=self.assess_data_quality(model_id, period),
            usage=self.assess_usage(model_id, period),
            alerts=self.generate_alerts(model_id, period),
            recommendations=self.generate_recommendations(model_id, period),
        )

    def assess_performance(self, model_id: str, period: str) -> PerformanceMetrics:
        """Assess model performance against expected benchmarks."""
        model = self.get_model(model_id)
        actuals = self.get_actual_outcomes(model_id, period)
        predictions = self.get_model_predictions(model_id, period)

        return PerformanceMetrics(
            accuracy=self.calculate_accuracy(actuals, predictions),
            precision=self.calculate_precision(actuals, predictions),
            recall=self.calculate_recall(actuals, predictions),
            f1=self.calculate_f1(actuals, predictions),
            auc=self.calculate_auc(actuals, predictions),
            calibration=self.calculate_calibration(predictions),
            benchmark_comparison=self.compare_to_benchmark(model_id, period),
        )
```

### Outcome Analysis

**Backtesting**: Compare model predictions to actual outcomes.

**Benchmarking**: Compare model performance to alternative models or approaches.

**Expert review**: Have subject matter experts review model outputs for reasonableness.

## Model Inventory

Banking organizations must maintain a complete model inventory.

```
MODEL INVENTORY RECORD:
├── Model ID: unique identifier
├── Model name: descriptive name
├── Model type: classification (e.g., "LLM-based credit scoring")
├── Risk tier: Critical/High/Medium/Low
├── Model owner: named individual
├── Model developer: team/individual
├── Business line: which business uses this model
├── Purpose: what the model is used for
├── Status: Development/Testing/Production/Retired
├── Deployment date: when deployed to production
├── Last validation date: when last independently validated
├── Next re-validation due: when next validation is due
├── Key assumptions: summary of critical assumptions
├── Key limitations: summary of critical limitations
├── Input data: description of input data
├── Output: description of model output
├── Dependencies: other models or systems this model depends on
├── Downstream users: who uses this model's output
└── Related models: models that depend on this model's output
```

## Change Management

Any change to a model must be assessed for impact and may trigger re-validation.

### Types of Model Changes

| Change Type | Examples | Re-validation Required? |
|------------|----------|----------------------|
| **Material** | Methodology change, significant input data change, intended use change | YES |
| **Non-material** | Minor parameter tuning, performance optimization | MAYBE (impact assessment required) |
| **Administrative** | Owner change, documentation update | NO (update inventory) |

### LLM-Specific Changes

| Change | Type | Notes |
|--------|------|-------|
| Base model upgrade (GPT-4 to GPT-5) | Material | Full re-validation required |
| Fine-tuning with new data | Material | Full re-validation required |
| Prompt template change | Non-material | Targeted validation, outcome analysis |
| Temperature/parameter change | Non-material | Impact assessment, monitoring |
| System instruction update | Non-material | Targeted testing on key scenarios |
| New tool/API integration | Material | Full integration testing |

## Common Interview Questions

### Question 1: "Is an LLM used for customer service a model under SR 11-7?"

**Good answer**:
It depends on what the LLM is doing. If it's providing general information ("what are your branch hours?"), probably not. But if it's providing financial advice ("should I refinance my mortgage?"), making recommendations, or influencing customer financial decisions, then yes, it likely qualifies as a model. The key questions are: (1) does it produce quantitative estimates or decisions, or (2) does it support or influence model-related decisions? If either is yes, it's a model and needs full model risk management.

### Question 2: "How do you validate an LLM that was trained by a third party (e.g., OpenAI)?"

**Good answer structure**:
1. Acknowledge the challenge: You cannot validate the base model's training or internal methodology
2. Focus on what you CAN validate: input validation, output quality, performance on your specific use case
3. Implement comprehensive outcome analysis: test the model on your data, compare to benchmarks
4. Document all limitations: you don't control the base model, you can't verify training data quality
5. Implement compensating controls: human review, output filtering, continuous monitoring
6. Establish vendor oversight: monitor OpenAI's model changes, maintain fallback options
7. Classify appropriately: third-party models may need higher risk classification due to reduced visibility

### Question 3: "How often should an LLM model be re-validated?"

**Good answer**:
Per SR 11-7, at least annually for high-risk models. But for LLMs specifically, re-validation should also be triggered by: (1) any change to the base model by the vendor, (2) fine-tuning with new data, (3) significant changes in input data distribution, (4) performance degradation detected by monitoring, (5) changes in the model's intended use, (6) regulatory requirement changes, or (7) a significant model-related incident. In practice, LLMs used in production banking should be monitored continuously and formally re-validated at least annually.

## Compliance Checklist

### Model Development
- [ ] Model purpose and intended use documented
- [ ] Model methodology documented
- [ ] Key assumptions and limitations identified
- [ ] Input data requirements documented
- [ ] Model tested on holdout data
- [ ] Model tested across relevant subsegments
- [ ] Model tested under stress scenarios
- [ ] Model documentation complete (model card)

### Model Validation
- [ ] Independent validation completed before production use
- [ ] Conceptual soundness assessed
- [ ] Implementation accuracy verified
- [ ] Outcome analysis completed
- [ ] Validation findings documented
- [ ] Remediation actions tracked and completed
- [ ] Model approved for production use

### Model Inventory
- [ ] Model registered in inventory
- [ ] Risk tier assigned
- [ ] Model owner designated
- [ ] All required fields populated
- [ ] Inventory updated for any changes

### Ongoing Monitoring
- [ ] Performance monitoring implemented
- [ ] Stability monitoring implemented
- [ ] Input data quality monitoring implemented
- [ ] Alerting thresholds defined
- [ ] Outcome analysis performed regularly
- [ ] Benchmarking against alternatives
- [ ] Monitoring reports reviewed by model owner

### Change Management
- [ ] Change management process documented
- [ ] Impact assessment for all model changes
- [ ] Material changes trigger re-validation
- [ ] Change history maintained
- [ ] Inventory updated for changes
