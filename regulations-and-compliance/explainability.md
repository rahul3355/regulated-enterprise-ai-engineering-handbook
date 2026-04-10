# AI Explainability

## What Is Explainability and Why It Matters

AI explainability is the ability to understand and explain how an AI system makes decisions. In banking, explainability is not just a technical nice-to-have -- it is a regulatory, legal, and ethical requirement.

**Regulatory drivers**:
- **GDPR** Article 22: Right to meaningful information about automated decision logic
- **ECOA/Regulation B** (US): Adverse action notices with specific reasons
- **FCA** Consumer Duty: Consumer understanding requirement
- **EU AI Act**: Transparency requirements for high-risk AI
- **SR 11-7**: Model documentation and understanding
- **BCBS 239**: Ability to trace reports to source data

**Why engineers must care**: If you cannot explain why your AI made a decision, the bank cannot use that decision for anything that affects customers. Period.

## Types of Explainability

### Global Explainability

Understanding the model's overall behavior: "How does the model generally make decisions?"

**Techniques**:
- Feature importance (global)
- Partial dependence plots
- Accumulated Local Effects (ALE) plots
- Model architecture documentation
- Training data analysis

**Example**: "The credit scoring model primarily considers payment history (35% weight), credit utilization (25% weight), and credit history length (15% weight)."

### Local Explainability

Understanding individual predictions: "Why did the model make THIS decision for THIS person?"

**Techniques**:
- SHAP (SHapley Additive exPlanations)
- LIME (Local Interpretable Model-agnostic Explanations)
- Counterfactual explanations
- Saliency maps (for neural networks)
- Attention visualization (for LLMs)

**Example**: "This customer's application was declined because: (1) 3 late payments in the last 12 months (-45 points), (2) credit utilization at 89% (-30 points), (3) no credit history older than 2 years (-15 points)."

## Explainability by Model Type

### Traditional Models

| Model Type | Explainability Level | Technique |
|-----------|---------------------|-----------|
| Linear regression | High | Coefficients directly interpretable |
| Logistic regression | High | Coefficients show feature impact |
| Decision trees | High | Tree path shows decision logic |
| Random forest | Medium | Feature importance, tree paths |
| Gradient boosting | Medium | Feature importance, SHAP values |

### Deep Learning Models

| Model Type | Explainability Level | Technique |
|-----------|---------------------|-----------|
| Feed-forward neural network | Low | SHAP, LIME, saliency maps |
| CNN | Low-Medium | Grad-CAM, feature visualization |
| RNN/LSTM | Low | Attention visualization, SHAP |
| Transformer/LLM | Low | Attention maps, SHAP, counterfactuals |

### LLM/GenAI Explainability

LLMs present unique explainability challenges because:
- Billions of parameters cannot be individually interpreted
- Attention mechanisms are necessary but not sufficient for explanation
- Emergent behaviors are not fully understood
- Outputs are non-deterministic (temperature > 0)

**Practical approach for LLMs**:

```
LLM EXPLAINABILITY STRATEGY:
├── Input-level explanation:
│   ├── What information was provided to the model?
│   ├── What instructions/prompts were given?
│   └── What context/references were included?
├── Output-level explanation:
│   ├── What did the model say?
│   ├── What sources/references does it cite?
│   └── What is the confidence level?
├── Process-level explanation:
│   ├── What model version was used?
│   ├── What temperature/parameters were set?
│   ├── What tools/APIs were called?
│   └── What post-processing was applied?
├── Outcome-level explanation:
│   ├── How does the output relate to the input?
│   ├── What would change the output?
│   └── What are the limitations of this output?
└── Human-level explanation:
    ├── What was the human reviewer's assessment?
    ├── Did they agree with the output?
    └── What modifications did they make?
```

## Techniques in Detail

### SHAP (SHapley Additive exPlanations)

Based on cooperative game theory, SHAP values explain individual predictions by assigning each feature an importance value.

```python
import shap
import xgboost

# Train model
model = xgboost.XGBClassifier().fit(X_train, y_train)

# Create explainer
explainer = shap.TreeExplainer(model)

# Get SHAP values for predictions
shap_values = explainer.shap_values(X_test)

# Explain individual prediction
def explain_prediction(model, shap_explainer, sample, feature_names):
    """Generate explanation for a single prediction."""
    prediction = model.predict(sample)[0]
    probability = model.predict_proba(sample)[0]
    shap_vals = shap_explainer.shap_values(sample)

    # Top contributing features
    top_features = sorted(
        zip(feature_names, shap_vals[0]),
        key=lambda x: abs(x[1]),
        reverse=True,
    )[:5]

    return Explanation(
        prediction=prediction,
        probability=float(max(probability)),
        base_value=float(shap_explainer.expected_value),
        feature_contributions=[
            FeatureContribution(
                feature=name,
                value=float(value),
                feature_value=float(sample[0][feature_names.index(name)]),
            )
            for name, value in top_features
        ],
        summary=f"This application was {'approved' if prediction == 1 else 'declined'} "
                f"with {max(probability):.0%} confidence. "
                f"Top factors: {', '.join(f'{name} ({value:+.1f})' for name, value in top_features[:3])}.",
    )
```

### Counterfactual Explanations

Answer: "What would need to change for a different outcome?"

```python
class CounterfactualExplainer:
    """Generate counterfactual explanations for model decisions."""

    def explain(self, model, sample, feature_names, target_change: str):
        """Find minimum changes to achieve different outcome."""
        current_prediction = model.predict(sample)[0]

        # Try changing one feature at a time
        for i, feature in enumerate(feature_names):
            original_value = sample[0][i]

            # For numerical features, try incremental changes
            if isinstance(original_value, (int, float)):
                for delta in self.get_feature_deltas(feature, original_value):
                    modified = sample.copy()
                    modified[0][i] = original_value + delta
                    new_prediction = model.predict(modified)[0]
                    if new_prediction != current_prediction:
                        return Counterfactual(
                            feature=feature,
                            current_value=original_value,
                            required_value=original_value + delta,
                            change=f"Change {feature} from {original_value} to {original_value + delta}",
                            confidence=self.estimate_confidence(model, modified),
                        )

            # For categorical features, try other categories
            if isinstance(original_value, str):
                for category in self.get_valid_categories(feature):
                    if category != original_value:
                        modified = sample.copy()
                        modified[0][i] = category
                        new_prediction = model.predict(modified)[0]
                        if new_prediction != current_prediction:
                            return Counterfactual(
                                feature=feature,
                                current_value=original_value,
                                required_value=category,
                                change=f"Change {feature} from '{original_value}' to '{category}'",
                                confidence=self.estimate_confidence(model, modified),
                            )

        return Counterfactual(
            feature=None,
            change="No single-feature change achieves different outcome",
            confidence=0.0,
        )
```

**Consumer-friendly counterfactual**: "If your credit card utilization were below 30% (currently 89%), your application would likely be approved. Alternatively, if you had no late payments in the last 12 months (currently 3), your application would likely be approved."

## Adverse Action Notices

Under ECOA/Regulation B (US) and similar regulations globally, when a credit application is declined, the bank must provide specific reasons.

### Requirements

```
ADVERSE ACTION NOTICE REQUIREMENTS:
├── Statement of action taken (declined)
├── Specific reasons for the action
├── Number of reasons (typically 4)
├── Reasons must be specific (not "internal policy")
├── Reasons must be accurate (actually influenced the decision)
├── Contact information for the creditor
├── Notice of right to request specific reasons
├── ECOA notice (your right to receive a copy of appraisal)
└── CFPB contact information
```

### AI-Specific Challenges

```
CHALLENGES FOR AI-DRIVEN ADVERSE ACTION:
├── Model features may not map to understandable reasons
│   Example: "embedding_dim_47" is not a valid reason
├── Non-linear models may have interaction effects
│   Example: "A + B together caused decline, but neither alone"
├── LLM outputs may not have traceable reasons
│   Example: "The AI said no but we don't know exactly why"
└── Model updates may change reason mappings
    Example: "Yesterday it was utilization, today it's payment history"

SOLUTIONS:
├── Map model features to reason codes
│   - Create a mapping: feature -> consumer-friendly reason
│   - Validate mapping with compliance team
│   - Update mapping when model changes
├── Use SHAP values to rank feature contributions
│   - Top contributing features become adverse action reasons
│   - Validate reasons are accurate and specific
├── Implement fallback for unexplainable models
│   - If model cannot produce valid reasons, require human review
│   - Document why the model is unexplainable
└── Test reasons for accuracy
    - Verify reasons actually influenced the decision
    - Verify reasons are not proxies for protected characteristics
```

## Explainability Documentation

### Model Explanation Document

```
MODEL EXPLANATION DOCUMENT:
├── Model overview:
│   ├── Model type and architecture
│   ├── Training data description
│   ├── Key features and their meanings
│   └── Known limitations
├── Global explanation:
│   ├── Feature importance ranking
│   ├── Feature direction (positive/negative impact)
│   ├── Interaction effects
│   └── Partial dependence plots
├── Local explanation methodology:
│   ├── Technique used (SHAP, LIME, etc.)
│   ├── How to interpret explanations
│   └── Limitations of explanations
├── Adverse action reason mapping:
│   ├── Feature-to-reason code mapping
│   ├── Reason code descriptions (consumer-friendly)
│   └── Validation of reason accuracy
├── Counterfactual capabilities:
│   ├── What counterfactuals can be generated
│   ├── How to interpret counterfactuals
│   └── Limitations
├── Example explanations:
│   ├── Approved application example
│   ├── Declined application example
│   └── Edge case examples
└── Validation:
    ├── Explanation accuracy assessment
    ├── Explanation stability testing
    └── User comprehension testing
```

## Explainability Testing

### How to Test Explanations

```
EXPLAINABILITY TEST SUITE:
├── Accuracy tests:
│   ├── Do explanations accurately reflect model behavior?
│   ├── If you follow the explanation's advice, does the outcome change?
│   └── Are explanations consistent across similar inputs?
├── Completeness tests:
│   ├── Do explanations cover all significant factors?
│   └── Are interaction effects captured?
├── Stability tests:
│   ├── Do similar inputs produce similar explanations?
│   └── Do explanations change significantly with small input changes?
├── Comprehensibility tests:
│   ├── Can users understand the explanations?
│   ├── Can users act on the explanations?
│   └── Do explanations help users improve outcomes?
└── Fairness tests:
    ├── Are explanations equally accurate across demographic groups?
    └── Do explanations reveal proxy discrimination?
```

## Common Interview Questions

### Question 1: "How do you explain a deep neural network's decision to a customer?"

**Good answer structure**:
I use a layered approach: (1) Start with the top-level factors using SHAP values -- "Your application was declined primarily due to X, Y, and Z." (2) Provide counterfactuals -- "If X were improved to [specific value], the outcome would likely change." (3) Avoid technical details about the neural network architecture -- the customer cares about what they can do, not how the model works internally. (4) Validate that explanations are accurate and specific -- generic explanations like "internal policy" are not acceptable. (5) If the model cannot produce valid explanations, escalate to human review.

### Question 2: "Is an LLM inherently explainable?"

**Good answer**:
No. LLMs with billions of parameters are inherently opaque. Attention mechanisms show which tokens the model focuses on, but this is not a complete explanation of the model's reasoning. For banking use cases, I would not rely on the LLM's internal explainability. Instead, I would use input-output explainability: what went in, what came out, what instructions were given, and what post-processing was applied. For high-stakes decisions, I would not use LLMs as the sole decision-maker -- they would be part of a broader system where explainable models make the actual decision, and the LLM assists with analysis or communication.

### Question 3: "How do you ensure adverse action reasons from an ML model are compliant?"

**Good answer structure**:
1. Map model features to regulatory-compliant reason codes (with legal review)
2. Use SHAP values to identify the top contributing features for each decision
3. Validate that reasons are specific, accurate, and actionable
4. Test reasons for fairness -- ensure they are not proxies for protected characteristics
5. Implement a fallback: if the model cannot produce valid reasons, require human review
6. Update the reason code mapping whenever the model changes
7. Document the entire process for auditors
