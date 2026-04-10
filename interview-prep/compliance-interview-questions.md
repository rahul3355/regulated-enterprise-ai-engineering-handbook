# Compliance Interview Questions (20+)

## Foundational Compliance

### Q1: What is the role of a compliance function in a bank?

**Strong Answer**: "The compliance function ensures the bank operates within legal and regulatory requirements. Key responsibilities: (1) **Regulatory monitoring** -- track changes in laws and regulations. (2) **Policy development** -- create internal policies that implement regulatory requirements. (3) **Training** -- educate employees on compliance obligations. (4) **Monitoring and testing** -- verify that business units follow policies. (5) **Reporting** -- file required reports to regulators (SARs, CTRs, Call Reports). (6) **Advisory** -- advise business units on compliance implications of new products, processes, and systems. (7) **Exam management** -- coordinate with regulatory examiners during audits. For technology teams, compliance means: building systems that enforce policies (not just document them), maintaining audit trails, and ensuring data accuracy for regulatory reporting."

### Q2: What is a regulatory examination and how should technology teams prepare?

**Strong Answer**: "Regulatory exams are periodic reviews by agencies (OCC, Fed, CFPB, state regulators) to assess the bank's compliance with laws. Technology teams prepare by: (1) **Documentation** -- maintain up-to-date system architecture diagrams, data flow diagrams, access control matrices, and change management records. (2) **Audit trails** -- ensure all systems log who did what, when, and the result. Logs must be tamper-proof and retained for the required period. (3) **Access reviews** -- demonstrate that access permissions are reviewed periodically and that terminated employees' access is revoked promptly. (4) **Incident records** -- show how security incidents and system failures were handled. (5) **Testing evidence** -- provide results of security testing, disaster recovery tests, and compliance monitoring. The key is maintaining compliance continuously, not scrambling when an exam is announced."

### Q3: What are the key compliance requirements for AI/ML systems in banking?

**Strong Answer**: "While AI-specific regulations are still evolving, existing frameworks apply: (1) **Fair lending** (ECOA/Regulation B) -- AI models used in lending must not discriminate against protected classes. This means testing for disparate impact and maintaining model documentation. (2) **Explainability** -- if an AI system denies a loan, the bank must provide specific reasons (adverse action notice). Black-box models make this difficult. (3) **Model risk management** (SR 11-7) -- all models, including ML models, must be validated, tested, and monitored. (4) **Data governance** -- training data must be accurate, complete, and appropriately sourced. (5) **Audit trail** -- AI decisions must be traceable to input data and model logic. (6) **Consumer protection** -- AI interactions with customers must be truthful and not misleading. The CFPB has indicated that existing consumer protection laws apply to AI-enabled activities."

## Data Governance

### Q4: What is data lineage and why is it important for compliance?

**Strong Answer**: "Data lineage tracks data from its origin through every transformation to its final destination. It answers: where did this data come from, what transformations were applied, who accessed it, and where did it end up? For compliance: (1) **Regulatory reporting** -- regulators want to know how reported numbers were derived. (2) **Error investigation** -- when a report is wrong, lineage helps trace the error to its source. (3) **Impact analysis** -- when a source system changes, lineage shows which downstream reports are affected. (4) **Audit evidence** -- demonstrates data integrity controls. In a GenAI context, data lineage for RAG means: which source documents were used, when were they indexed, what version was retrieved, and how was the response generated."

### Q5: What is GDPR and how does it affect banking technology?

**Strong Answer**: "GDPR (General Data Protection Regulation) is EU data privacy law that applies to any organization processing EU residents' data. Key requirements affecting technology: (1) **Right to access** -- customers can request all their personal data. Systems must be able to extract and provide this. (2) **Right to erasure** -- customers can request deletion of their data. Systems must support complete deletion across all storage. (3) **Data minimization** -- collect only data that is necessary. (4) **Purpose limitation** -- use data only for the purpose it was collected. (5) **Consent management** -- track and manage customer consent for data processing. (6) **Data breach notification** -- notify regulators within 72 hours of a breach. For global banks, GDPR compliance often becomes the baseline for all data handling, even for non-EU customers."

## Model Compliance

### Q6: What is SR 11-7 (Model Risk Management guidance)?

**Strong Answer**: "SR 11-7 is a joint guidance from the Federal Reserve and OCC on model risk management. It applies to all quantitative models used in banking, including ML/AI models. Key requirements: (1) **Model inventory** -- maintain a complete list of all models in use. (2) **Model validation** -- independent validation before deployment and periodically thereafter. (3) **Ongoing monitoring** — track model performance and detect degradation. (4) **Documentation** -- comprehensive documentation of model methodology, data, assumptions, and limitations. (5) **Governance** -- clear roles and responsibilities for model development, validation, and use. (6) **Internal controls** -- ensure models are used appropriately and their outputs are interpreted correctly. For GenAI: any AI system that makes or supports decisions (credit scoring, fraud detection, compliance advice) falls under SR 11-7 and must follow these requirements."

### Q7: How do you validate an AI model for production use in banking?

**Strong Answer**: "Model validation follows a structured process: (1) **Conceptual soundness** -- is the model appropriate for the intended use? Are the assumptions reasonable? (2) **Data quality** -- is the training data representative, accurate, and complete? Are there biases? (3) **Performance testing** -- how accurate is the model on held-out test data? What are the false positive/negative rates? (4) **Stress testing** -- how does the model perform under edge cases and adversarial conditions? (5) **Fairness testing** -- does the model produce disparate outcomes for different demographic groups? (6) **Explainability** -- can the model's decisions be explained? (7) **Ongoing monitoring plan** -- how will we detect model degradation in production? For GenAI models, validation also includes: hallucination rate, groundedness score, prompt injection resistance, and output consistency."

## Operational Compliance

### Q8: What is SOX compliance and how does it affect IT systems?

**Strong Answer**: "Sarbanes-Oxley (SOX) requires public companies to maintain accurate financial reporting and internal controls. For IT systems: (1) **Access controls** -- who can modify financial data and systems? Access must be restricted and logged. (2) **Change management** -- all changes to financial systems must be approved, tested, and documented. (3) **Segregation of duties** -- the person who develops code should not be the same person who deploys it to production. (4) **Audit trails** -- all financial transactions and system changes must be logged. (5) **IT general controls** -- password policies, backup procedures, disaster recovery plans. (6) **Third-party risk** -- vendors that process financial data must be assessed. For GenAI systems that generate financial reports or support financial decisions, SOX compliance means: the AI's outputs must be auditable, the system must have change controls, and access to the AI system must be restricted."

### Q9: What is a compliance risk assessment?

**Strong Answer**: "A compliance risk assessment identifies, evaluates, and prioritizes compliance risks across the organization. Process: (1) **Identify risks** -- what regulations apply? What could go wrong? (2) **Assess likelihood** -- how probable is each risk? (3) **Assess impact** -- what would be the consequences (fines, reputation, operational)? (4) **Evaluate controls** -- what controls exist to mitigate each risk? (5) **Calculate residual risk** -- likelihood x impact after controls. (6) **Prioritize** -- focus on highest residual risks. (7) **Action plan** -- what additional controls are needed? For technology teams, this means: every new system or significant change triggers a compliance risk assessment, and the results determine what controls must be built in."

## Scenario-Based Compliance Questions

### Q10: A new regulation is published that affects how your AI system operates. What do you do?

**Strong Answer**: "Immediate steps: (1) **Assess impact** -- work with the compliance team to understand exactly what the regulation requires and how it affects our system. (2) **Gap analysis** -- compare current system behavior with regulatory requirements. (3) **Risk assessment** -- how urgent is compliance? What is the penalty for non-compliance? (4) **Action plan** -- what changes are needed to the AI system? Do we need new controls, different model behavior, additional logging? (5) **Implementation** -- make the changes following our change management process (development, testing, approval, deployment). (6) **Testing** -- verify the changes meet regulatory requirements. (7) **Documentation** -- update system documentation and compliance records. (8) **Training** -- if the changes affect how users interact with the system, provide training. The key is having a process for regulatory change management that is tested and practiced."

### Q11: An auditor finds that your AI system provided incorrect information to a customer. What happens next?

**Strong Answer**: "Immediate response: (1) **Contain** -- determine if this is an isolated incident or systemic issue. If systemic, pause the affected feature. (2) **Investigate** -- what caused the incorrect information? Was it a retrieval failure, generation error, or outdated source document? (3) **Assess impact** -- how many customers were affected? What decisions did they make based on incorrect information? (4) **Remediate** -- fix the root cause and correct any customer harm (reverse incorrect fees, provide correct information). (5) **Report** -- inform compliance, legal, and potentially regulators depending on the severity. (6) **Prevent** -- implement additional controls (verification, human review, monitoring) to prevent recurrence. (7) **Document** -- maintain a complete record of the incident and remediation for the auditor. The key is treating this as a learning opportunity -- each incident reveals a gap in controls that can be closed."

## Quick-Fire Compliance Questions

### Q12: What is a SAR filing deadline?
**Answer**: "30 calendar days from initial detection of suspicious activity. Can be extended to 60 days if more time is needed to identify a suspect."

### Q13: What is the Bank Secrecy Act?
**Answer**: "Federal law requiring financial institutions to assist government agencies in detecting and preventing money laundering. Requires CTRs, SARs, and customer identification programs."

### Q14: What is a CTR (Currency Transaction Report)?
**Answer**: "A report filed for cash transactions exceeding $10,000 in a single business day. Filed with FinCEN within 15 days."

### Q15: What is OFAC?
**Answer**: "Office of Foreign Assets Control. Administers and enforces economic and trade sanctions. Banks must screen customers and transactions against OFAC sanctions lists."

### Q16: What is the CFPB?
**Answer**: "Consumer Financial Protection Bureau. Regulates consumer financial products and services, enforces consumer protection laws, and handles consumer complaints."

### Q17: What is adverse action notice?
**Answer**: "A notice required by ECOA/Regulation B when a creditor denies credit. Must include specific reasons for the denial."

### Q18: What is fair lending?
**Answer**: "The principle that credit decisions must not discriminate based on protected characteristics (race, gender, age, religion, national origin, marital status). Enforced by ECOA and Fair Housing Act."

### Q19: What is a compliance management system (CMS)?
**Answer**: "A structured framework for managing compliance risk. Includes: board oversight, compliance policies, training, monitoring, complaint resolution, and corrective action."

### Q20: What is the difference between compliance and risk management?
**Answer**: "Compliance ensures adherence to laws and regulations. Risk management identifies, assesses, and mitigates all types of risks (credit, market, operational, reputational). Compliance is a subset of risk management."
