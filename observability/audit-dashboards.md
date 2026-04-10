# Compliance Audit Dashboards for Banking GenAI

## Purpose

Compliance audit dashboards provide evidence that the GenAI platform operates within regulatory requirements. Unlike operational dashboards (which are for engineers), audit dashboards serve compliance officers, internal auditors, and external regulators.

## Key Regulatory Frameworks

```
┌─────────────────────────────────────────────────────────────┐
│  REGULATORY FRAMEWORKS AFFECTING GenAI IN BANKING            │
│                                                             │
│  GLBA (Gramm-Leach-Bliley Act)                              │
│    - Protect customer financial data                        │
│    - Dashboard: PII in logs, data access tracking            │
│                                                             │
│  ECOA (Equal Credit Opportunity Act)                        │
│    - Fair lending, no discrimination                        │
│    - Dashboard: AI decision fairness across demographics     │
│                                                             │
│  TILA (Truth in Lending Act)                                │
│    - Accurate lending disclosures                           │
│    - Dashboard: Disclaimer compliance, accurate rate quotes  │
│                                                             │
│  SOX (Sarbanes-Oxley)                                       │
│    - Audit trails for financial reporting                   │
│    - Dashboard: Complete audit trail coverage               │
│                                                             │
│  CCPA/CPRA (California Consumer Privacy Act)                │
│    - Data privacy, right to delete                          │
│    - Dashboard: Data subject request compliance              │
│                                                             │
│  SR 11-7 (Model Risk Management)                            │
│    - Model governance and validation                        │
│    - Dashboard: Model inventory, validation status           │
└─────────────────────────────────────────────────────────────┘
```

## Audit Dashboard Categories

### 1. Data Privacy Compliance

```
┌──────────────────────────────────────────────────────────────┐
│  DATA PRIVACY COMPLIANCE                                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PII Detection in Logs (last 30 days):                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  SSN patterns detected:        0       [PASS]          │ │
│  │  Credit card patterns:         0       [PASS]          │ │
│  │  Email addresses in logs:      0       [PASS]          │ │
│  │  Account numbers in logs:      0       [PASS]          │ │
│  │  Phone numbers in logs:        0       [PASS]          │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Prompt Redaction Effectiveness:                             │
│  Total prompts processed:     1,245,678                     │
│  Prompts with sensitive data: 89,234 (7.2%)                 │
│  Successfully redacted:       89,234 (100%) [PASS]          │
│  Redaction failures:          0       [PASS]                │
│                                                              │
│  Data Retention Compliance:                                  │
│  Audit logs within retention: 100%    [PASS]                │
│  Expired logs purged:         45,678  [PASS]                │
│  Over-retention violations:   0       [PASS]                │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 2. AI Decision Audit Trail

```
┌──────────────────────────────────────────────────────────────┐
│  AI DECISION AUDIT TRAIL                                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Audit Trail Coverage:                                       │
│  Total AI interactions:        1,245,678                     │
│  With complete audit entry:    1,245,678 (100%) [PASS]       │
│  Missing audit entries:        0             [PASS]           │
│                                                              │
│  Audit Entry Completeness:                                   │
│  Model version logged:         100%   [PASS]                │
│  Prompt metadata logged:       100%   [PASS]                │
│  Response metadata logged:     100%   [PASS]                │
│  Source documents logged:      100%   [PASS]                │
│  Compliance disclaimer logged: 100%   [PASS]                │
│  Timestamp present:            100%   [PASS]                │
│  Correlation ID present:       100%   [PASS]                │
│                                                              │
│  Audit Log Integrity:                                        │
│  Tamper-evident hash verified: 100%   [PASS]                │
│  Chain of custody intact:      100%   [PASS]                │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 3. Model Governance

```
┌──────────────────────────────────────────────────────────────┐
│  MODEL GOVERNANCE                                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Model Inventory:                                            │
│  ┌────────────┬─────────┬────────────┬────────┬────────────┐ │
│  │ Model      │ Version │ Status     │ Last   │ Next       │ │
│  │            │         │            │ Valid  │ Review     │ │
│  ├────────────┼─────────┼────────────┼────────┼────────────┤ │
│  │ gpt-4      │ v2024.1 │ Approved   │ 03/01  │ 06/01      │ │
│  │ gpt-3.5    │ v2024.1 │ Approved   │ 03/01  │ 06/01      │ │
│  │ claude-3   │ v1.0    │ Approved   │ 02/15  │ 05/15      │ │
│  │ embedding  │ v3-lg   │ Approved   │ 03/10  │ 06/10      │ │
│  │ reranker   │ v2.1    │ Pending    │ 03/12  │ 03/20      │ │
│  └────────────┴─────────┴────────────┴────────┴────────────┘ │
│                                                              │
│  Model Change Log (last 90 days):                            │
│  2025-03-01: gpt-4 upgraded to 2024.1 (approved by ML board)│
│  2025-02-15: claude-3 added to production (approved)         │
│  2025-01-20: gpt-3.5-turbo-0125 deprecated (migrated)        │
│                                                              │
│  Model Risk Assessment:                                      │
│  High-risk models (financial advice): 3                      │
│  All high-risk models have documented validation: YES        │
│  All high-risk models have rollback plan: YES                │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 4. Fairness and Bias Monitoring

```
┌──────────────────────────────────────────────────────────────┐
│  AI FAIRNESS MONITORING                                        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Response Quality by Demographic Segment*:                   │
│  ┌──────────────┬───────────┬───────────┬───────────────────┐│
│  │ Segment      │ Avg Score │ Halluc.   │ Satisfaction      ││
│  ├──────────────┼───────────┼───────────┼───────────────────┤│
│  │ Age 18-30    │ 0.91      │ 2.1%      │ 4.1/5.0           ││
│  │ Age 31-50    │ 0.92      │ 1.9%      │ 4.2/5.0           ││
│  │ Age 51+      │ 0.90      │ 2.3%      │ 4.0/5.0           ││
│  │ Group A      │ 0.91      │ 2.0%      │ 4.1/5.0           ││
│  │ Group B      │ 0.92      │ 2.1%      │ 4.2/5.0           ││
│  │ Group C      │ 0.91      │ 2.2%      │ 4.1/5.0           ││
│  └──────────────┴───────────┴───────────┴───────────────────┘│
│                                                              │
│  Disparity Alert Threshold: > 5% difference between segments │
│  Current max disparity: 2.2% (within threshold) [PASS]       │
│                                                              │
│  * Demographic data is aggregated and anonymized.            │
│    Individual-level data is never stored in audit dashboards. │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 5. Compliance Checklist Status

```
┌──────────────────────────────────────────────────────────────┐
│  QUARTERLY COMPLIANCE CHECKLIST - Q1 2025                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Data Protection:                                            │
│  [x] PII redaction in all logs verified                      │
│  [x] Data encryption at rest confirmed                       │
│  [x] Access logs reviewed for unauthorized access            │
│  [x] Data retention policy enforced                          │
│  [x] Data subject requests processed within 30 days          │
│                                                              │
│  AI Governance:                                              │
│  [x] All production models have valid approval               │
│  [x] Model risk assessments current (< 90 days)              │
│  [x] Model change log complete and accurate                  │
│  [x] Rollback procedures documented and tested               │
│  [x] Training data provenance documented                     │
│                                                              │
│  Operational Compliance:                                     │
│  [x] All AI decisions have audit trail entries               │
│  [x] Compliance disclaimers shown for all advice             │
│  [x] Human escalation path available and tested              │
│  [x] Incident response procedures current                    │
│  [x] Staff training on AI governance completed               │
│                                                              │
│  Overall Status: 20/20 items compliant (100%)                │
│  Last audit date: 2025-03-10                                 │
│  Next audit date: 2025-06-10                                 │
│  Auditor: Internal Compliance Team                           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## Generating Audit Reports

```python
def generate_quarterly_audit_report(quarter: str, year: int) -> dict:
    """Generate a comprehensive quarterly audit report."""
    return {
        'report_id': f"AUDIT-{year}-Q{quarter}",
        'period': {
            'start': f"{year}-{(quarter-1)*3+1:02d}-01",
            'end': f"{year}-{min(quarter*3, 12):02d}-28",
        },
        'generated_at': datetime.utcnow().isoformat(),
        'generated_by': 'automated-audit-system',
        'sections': {
            'data_privacy': collect_privacy_metrics(),
            'audit_trail': collect_audit_trail_metrics(),
            'model_governance': collect_model_governance_metrics(),
            'fairness': collect_fairness_metrics(),
            'compliance_checklist': evaluate_compliance_checklist(),
            'incidents': collect_incident_summary(),
            'remediation': collect_remediation_status(),
        },
        'overall_status': determine_overall_status(),
        'recommendations': generate_recommendations(),
        'attestation': {
            'prepared_by': 'Compliance Engineering Team',
            'reviewed_by': 'Chief Compliance Officer',
            'date': datetime.utcnow().date().isoformat(),
        }
    }
```

## Dashboard Access Control

```
┌─────────────────────────────────────────────────────────────┐
│  AUDIT DASHBOARD ACCESS CONTROL                              │
├───────────────────────┬─────────────────────────────────────┤
│  Role                 │  Access                             │
├───────────────────────┼─────────────────────────────────────┤
│  Compliance Officer   │  Full access, all dashboards        │
│  Internal Auditor     │  Full access, read-only             │
│  External Regulator   │  Read-only, specific dashboards     │
│  Engineering Manager  │  Technical dashboards only          │
│  Engineer             │  No access to audit dashboards      │
│                       │  (use operational dashboards)       │
└───────────────────────┴─────────────────────────────────────┘

All access to audit dashboards is itself logged and audited.
```

## Common Audit Dashboard Mistakes

1. **Only preparing for audits quarterly**: Audit readiness should be continuous. A dashboard that is built the week before an audit is likely to reveal surprises.

2. **Including PII in audit data**: Even audit dashboards must respect privacy. Use aggregated, anonymized data.

3. **No historical comparison**: Regulators want to see trends. "100% compliant this quarter" is less meaningful than "100% compliant for 4 consecutive quarters."

4. **Missing remediation tracking**: Finding compliance gaps is expected. Failing to track their remediation is the real failure.

5. **Dashboard not self-documenting**: Every metric on an audit dashboard should have a clear definition, data source, and calculation method documented.

6. **No version control for dashboards**: Audit dashboard definitions should be version-controlled. Regulators may ask how a metric was calculated 2 years ago.
