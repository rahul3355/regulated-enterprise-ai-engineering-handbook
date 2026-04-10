# Compliance Terms

> Essential compliance and regulatory terminology for GenAI engineers in the banking sector. Each term includes definition, example, related concepts, and common misunderstandings.

## Glossary

### 1. GDPR (General Data Protection Regulation)
**Definition:** EU regulation governing data protection and privacy for individuals within the European Economic Area.
**Example:** A German employee exercises their "right to access" by requesting a copy of all their GenAI assistant interactions, which the bank must provide within 30 days.
**Related Concepts:** Data subject rights, DPO, data minimization, consent
**Common Misunderstanding:** GDPR applies to any organization processing EU residents' data, regardless of where the organization is located.

### 2. Data Subject
**Definition:** An identified or identifiable individual whose personal data is being processed.
**Example:** Every bank employee who uses the GenAI assistant is a data subject — their queries and interaction patterns are personal data.
**Related Concepts:** PII, data processing, consent, data subject rights
**Common Misunderstanding:** Data subjects are not just customers — employees, contractors, and vendors are also data subjects.

### 3. Data Controller
**Definition:** The entity that determines the purposes and means of processing personal data.
**Example:** The bank is the data controller for employee data processed through the GenAI assistant — it decides what data is collected and why.
**Related Concepts:** Data processor, DPA, joint controller
**Common Misunderstanding:** The data controller is not necessarily the entity that physically processes the data. The LLM provider is a data processor, not a controller.

### 4. Data Processor
**Definition:** An entity that processes personal data on behalf of the data controller.
**Example:** OpenAI is a data processor when the bank sends employee queries to its API — it processes data on the bank's behalf.
**Related Concepts:** Data controller, DPA, subprocessor
**Common Misunderstanding:** Data processors have legal obligations under GDPR too — they're not just passive pipes.

### 5. DPA (Data Processing Agreement)
**Definition:** A legally binding contract between a data controller and data processor that governs how personal data is processed.
**Example:** Before using the GenAI assistant, the bank must have a signed DPA with the LLM provider specifying data handling, deletion, and audit rights.
**Related Concepts:** GDPR Article 28, data controller, data processor
**Common Misunderstanding:** A DPA is not optional when personal data is shared with external processors. It's a GDPR requirement.

### 6. Data Minimization
**Definition:** The principle that only the minimum necessary personal data should be collected and processed for the stated purpose.
**Example:** The GenAI assistant sends only the query text and necessary context documents to the LLM — not the employee's full profile or access history.
**Related Concepts:** Purpose limitation, storage limitation, privacy by design
**Common Misunderstanding:** Data minimization doesn't mean "as little data as possible" — it means "no more than necessary for the purpose."

### 7. Right to Access (DSAR)
**Definition:** A data subject's right to obtain a copy of their personal data and information about how it's being processed.
**Example:** An employee submits a DSAR requesting all queries they've made to the GenAI assistant, responses received, and documents retrieved.
**Related Concepts:** GDPR Article 15, data portability, right to erasure
**Common Misunderstanding:** DSARs are not just about giving the data — they include explaining what the data is used for, who it's shared with, and how long it's retained.

### 8. Right to Erasure (Right to Be Forgotten)
**Definition:** A data subject's right to have their personal data deleted in certain circumstances.
**Example:** An employee who leaves the bank requests deletion of their GenAI interaction history. The bank must balance this against 7-year regulatory retention requirements.
**Related Concepts:** Right to access, data retention, legal hold
**Common Misunderstanding:** The right to erasure is not absolute. Legal and regulatory obligations (like banking record-keeping) can override it.

### 9. PII (Personally Identifiable Information)
**Definition:** Any data that can be used to identify an individual, directly or indirectly.
**Example:** Employee names, email addresses, query content, and even query patterns (time of day, topics of interest) can be PII.
**Related Concepts:** Personal data, sensitive data, anonymization, pseudonymization
**Common Misunderstanding:** PII isn't just obvious identifiers like names and SSNs. In combination, seemingly innocuous data can identify individuals.

### 10. PCI-DSS (Payment Card Industry Data Security Standard)
**Definition:** A security standard for organizations that handle credit card information.
**Example:** The GenAI assistant must never receive, process, or store credit card numbers in queries. If an employee asks "What's my credit card balance?", the system must redirect to the banking portal without processing card details.
**Related Concepts:** Cardholder data, tokenization, encryption, scope
**Common Misunderstanding:** PCI-DSS applies to ANY system that touches cardholder data — including a GenAI assistant if employees could paste card numbers into queries.

### 11. SOX (Sarbanes-Oxley Act)
**Definition:** US federal law requiring public companies to establish internal controls and procedures for financial reporting.
**Example:** If the GenAI assistant is used for financial reporting tasks, the system's controls, change management, and access logs must be auditable for SOX compliance.
**Related Concepts:** Internal controls, audit trail, segregation of duties
**Common Misunderstanding:** SOX is not just about the finance team — it affects any system involved in financial reporting or data.

### 12. Basel III/IV
**Definition:** International regulatory framework for bank capital requirements, leverage ratios, and liquidity standards.
**Example:** If the GenAI assistant provides information about capital requirements, the information must be accurate and sourced from current regulatory documents. Incorrect information could lead to compliance violations.
**Related Concepts:** Capital adequacy, risk-weighted assets, model risk management
**Common Misunderstanding:** Basel III is not a one-time regulation — it's continuously evolving, and banks must update their systems accordingly.

### 13. AI Governance
**Definition:** The framework of policies, procedures, and controls for the responsible development and use of AI systems.
**Example:** The bank's AI governance policy requires: model risk assessment, human oversight, bias testing, transparency disclosures, and regular audits for all GenAI systems.
**Related Concepts:** EU AI Act, model risk management, responsible AI, AI ethics
**Common Misunderstanding:** AI governance is not just ethics — it's a regulatory requirement with specific controls and documentation.

### 14. EU AI Act
**Definition:** European Union regulation classifying AI systems by risk level and imposing requirements accordingly.
**Example:** A GenAI assistant used for credit decisions would be classified as "high risk" under the AI Act, requiring conformity assessment, human oversight, and transparency.
**Related Concepts:** Risk classification, conformity assessment, prohibited AI practices
**Common Misunderstanding:** The AI Act applies to AI systems used in the EU, regardless of where the system is developed or deployed.

### 15. Model Risk Management (MRM)
**Definition:** The framework for identifying, measuring, and managing risks associated with the use of models in business decisions.
**Example:** The GenAI assistant used for compliance guidance is subject to MRM per SR 11-7: model validation, independent review, ongoing monitoring, and documentation.
**Related Concepts:** SR 11-7, model validation, model inventory, model governance
**Common Misunderstanding:** MRM is not just for quantitative models. GenAI systems that influence business decisions are also models under regulatory definitions.

### 16. SR 11-7 (OCC/Fed Guidance on Model Risk Management)
**Definition:** US regulatory guidance defining how banks should manage risk from models used in business decisions.
**Example:** The GenAI assistant's RAG pipeline is a "model" under SR 11-7 and must have: documented methodology, independent validation, ongoing performance monitoring, and change management controls.
**Related Concepts:** MRM, model validation, model inventory
**Common Misunderstanding:** SR 11-7 is guidance, not a law — but regulators enforce it as if it were. Non-compliance leads to supervisory actions.

### 17. Audit
**Definition:** An independent examination of systems, processes, or controls to verify compliance with requirements.
**Example:** An external auditor reviews the GenAI platform's access controls, audit logs, change management, and compliance procedures annually.
**Related Concepts:** Internal audit, external audit, regulatory examination, findings
**Common Misunderstanding:** Audits are not "gotcha" exercises. They're systematic assessments. Preparing for audits is an ongoing discipline, not a pre-audit scramble.

### 18. Regulatory Examination
**Definition:** A review by a regulatory body to assess a bank's compliance with applicable laws and regulations.
**Example:** The OCC examines the bank's GenAI systems as part of its annual safety and soundness examination, reviewing model risk management, consumer protection, and operational risk.
**Related Concepts:** Audit, enforcement action, consent order, findings
**Common Misunderstanding:** Regulatory examinations are not scheduled — they can happen at any time. Continuous readiness is required.

### 19. Data Retention
**Definition:** The policy governing how long data must be kept and when it must be destroyed.
**Example:** GenAI audit logs must be retained for 7 years per banking regulations. After 7 years, they must be securely destroyed unless subject to a legal hold.
**Related Concepts:** Data lifecycle, legal hold, right to erasure, storage costs
**Common Misunderstanding:** Data retention is not "keep everything forever." It's a balance of regulatory requirements, storage costs, and privacy obligations.

### 20. Legal Hold
**Definition:** A process that suspends normal data destruction when litigation or investigation is anticipated.
**Example:** During a regulatory investigation, all GenAI audit logs related to a specific employee's queries are placed under legal hold and cannot be deleted, even if past the retention period.
**Related Concepts:** Data retention, e-discovery, spoliation
**Common Misunderstanding:** Legal hold overrides normal retention policies. Deleting data under legal hold is spoliation — a serious legal violation.

### 21. Compliance Risk
**Definition:** The risk of legal or regulatory sanctions, financial loss, or reputational damage from failure to comply with laws and regulations.
**Example:** If the GenAI assistant provides incorrect compliance information that leads to a regulatory violation, the bank faces compliance risk (fines, sanctions, reputational damage).
**Related Concepts:** Operational risk, legal risk, reputational risk, risk appetite
**Common Misunderstanding:** Compliance risk is not just the fine amount — it includes remediation costs, increased regulatory scrutiny, and reputational damage.

### 22. KYC (Know Your Customer)
**Definition:** Regulatory requirements for verifying the identity of customers and assessing their risk profiles.
**Example:** If the GenAI assistant helps relationship managers with customer information, it must only surface KYC-approved data and must not suggest bypassing KYC procedures.
**Related Concepts:** AML, customer due diligence, identity verification
**Common Misunderstanding:** KYC is not a one-time process — it's ongoing monitoring throughout the customer relationship.

### 23. AML (Anti-Money Laundering)
**Definition:** Regulations and procedures designed to prevent criminals from disguising illegally obtained funds as legitimate income.
**Example:** The GenAI assistant must never suggest methods for circumventing AML reporting thresholds or structuring transactions to avoid detection.
**Related Concepts:** KYC, SAR (Suspicious Activity Report), CTR (Currency Transaction Report)
**Common Misunderstanding:** AML applies to all financial transactions, not just large cash deposits. Digital transfers and crypto are also covered.

### 24. Consent
**Definition:** Freely given, specific, informed, and unambiguous agreement to process personal data.
**Example:** Before the GenAI assistant collects employee interaction data for quality improvement, employees must be informed about what data is collected and how it's used, and must have the option to opt out.
**Related Concepts:** GDPR, privacy notice, legitimate interest, opt-in/opt-out
**Common Misunderstanding:** Consent must be as easy to withdraw as to give. Pre-checked boxes and buried terms are not valid consent.

### 25. Privacy by Design
**Definition:** An approach where privacy considerations are embedded into system design from the beginning, not added as an afterthought.
**Example:** The GenAI assistant is designed with: data minimization (only necessary data sent to LLM), PII redaction (automatic detection and masking), purpose limitation (data used only for query answering), and user transparency (clear privacy notices).
**Related Concepts:** Data minimization, privacy by default, DPIA
**Common Misunderstanding:** Privacy by Design is not a feature checklist. It's a methodology that influences every design decision from day one.
