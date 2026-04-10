# Banking Domain Interview Questions (30+)

## Foundational Banking Knowledge

### Q1: What are the main business lines of a typical retail bank?

**Strong Answer**: "The main retail banking lines are: (1) **Deposits** -- checking accounts, savings accounts, CDs, money market accounts. This is the bank's funding base. (2) **Lending** -- personal loans, mortgages, auto loans, credit cards. This is the primary revenue driver through interest income. (3) **Payments** -- wire transfers, ACH, debit cards, bill pay. Revenue from fees and interchange. (4) **Wealth Management** -- investment advisory, brokerage, retirement accounts. Revenue from management fees. (5) **Treasury Services** -- foreign exchange, interest rate products for institutional clients. In a GenAI context, each line has its own policies, regulations, and knowledge base."

### Q2: What is KYC and why is it important?

**Strong Answer**: "KYC (Know Your Customer) is the process of verifying the identity of clients and assessing their risk profiles. It involves collecting identification documents, understanding the nature of the customer's activities, and monitoring for suspicious behavior. It's important because: (1) **Regulatory requirement** -- mandated by the Bank Secrecy Act, USA PATRIOT Act, and similar regulations globally. (2) **Fraud prevention** -- prevents identity theft and account takeover. (3) **AML compliance** -- KYC is the foundation of anti-money laundering programs. (4) **Risk management** -- understanding customers helps assess credit and operational risk. Failure to maintain adequate KYC procedures can result in massive regulatory fines."

### Q3: Explain the difference between Regulation E and Regulation Z.

**Strong Answer**: "Regulation E implements the Electronic Fund Transfer Act and covers electronic transfers -- debit card transactions, ATM withdrawals, direct deposits, online bill payments. Key consumer protections include: liability limits for unauthorized transfers ($50 if reported within 2 days), error resolution procedures, and periodic statement requirements. Regulation Z implements the Truth in Lending Act and covers consumer credit -- mortgages, credit cards, personal loans. Key requirements include: APR disclosure, finance charge disclosure, right of rescission for home loans, and billing error resolution. Both are enforced by the CFPB. In a RAG system, these regulations would be separate documents with distinct sections and update cycles."

### Q4: What is the difference between a bank's front office, middle office, and back office?

**Strong Answer**: "**Front office** is revenue-generating and customer-facing: retail banking (branches, call centers), investment banking (deal origination), trading, sales. **Middle office** manages risk and ensures compliance: risk management, compliance, treasury, strategic planning. **Back office** handles operations and infrastructure: settlements, clearances, record maintenance, accounting, IT, HR. In banking technology, front office systems need low latency and good UX, middle office systems need accuracy and auditability, and back office systems need reliability and integration capabilities."

### Q5: What is AML and what are the key components of an AML program?

**Strong Answer**: "AML (Anti-Money Laundering) refers to laws and procedures designed to prevent criminals from disguising illegally obtained funds as legitimate income. Key components: (1) **Customer Due Diligence (CDD)** -- verify identity and assess risk. (2) **Enhanced Due Diligence (EDD)** -- extra scrutiny for high-risk customers. (3) **Transaction Monitoring** -- automated systems that flag suspicious patterns. (4) **Suspicious Activity Reports (SARs)** -- filed with FinCEN when suspicious activity is detected. (5) **Currency Transaction Reports (CTRs)** -- filed for cash transactions over $10,000. (6) **Sanctions Screening** -- check customers against OFAC lists. (7) **Independent Audit** -- regular testing of the AML program. A GenAI assistant helping with AML would need access to all these procedures and must be extremely accurate."

## Banking Technology

### Q6: What is a core banking system?

**Strong Answer**: "A core banking system is the back-end software that processes daily banking transactions and updates financial records. It handles: deposits, withdrawals, loans, interest calculations, customer accounts, and general ledger. Major vendors include FIS (Profile), Fiserv (Premier), Jack Henry (SilverLake), and Temenos (T24). Core systems are typically decades-old mainframe or legacy systems that are difficult to modernize. Banks layer modern APIs and digital channels on top of the core rather than replacing it. For GenAI applications, the core banking system is usually accessed via an integration layer (APIs, message queues) rather than direct database access."

### Q7: What is SWIFT and how does it work?

**Strong Answer**: "SWIFT (Society for Worldwide Interbank Financial Telecommunication) is a messaging network that banks use to securely transmit information about financial transactions, primarily international wire transfers. SWIFT doesn't move money -- it sends payment orders between banks. Each bank has a unique SWIFT/BIC code (8-11 characters). A SWIFT message contains: sender/receiver identification, transaction amount, currency, value date, and beneficiary details. SWIFT messages are standardized (MT/MX formats) and encrypted. For GenAI, understanding SWIFT is relevant when building assistants that help with international payments or trade finance."

### Q8: What is the difference between ACH and wire transfers?

**Strong Answer**: "**ACH (Automated Clearing House)** is a batch processing system for electronic payments. Transactions are grouped and processed in batches 2-3 times per day. Takes 1-3 business days. Low cost ($0.20-$1.50 per transaction). Used for payroll, bill payments, direct deposits. Reversible within certain windows. **Wire transfers** are real-time, individual transactions processed through Fedwire or CHIPS. Same-day or immediate settlement. Higher cost ($15-$50 per transaction). Used for large, time-sensitive payments. Generally irreversible once sent. In a customer service AI, understanding this distinction is critical for answering 'when will my payment arrive?' questions."

## Banking Regulations

### Q9: What is Dodd-Frank and how does it affect banking technology?

**Strong Answer**: "Dodd-Frank (2010) was enacted after the 2008 financial crisis to increase financial system stability. Key provisions affecting technology: (1) **Stress testing** -- banks must run regular stress tests requiring extensive data aggregation and reporting systems. (2) **Volcker Rule** -- limits proprietary trading, requiring trade monitoring systems. (3) **Consumer protection** -- CFPB oversight requires banks to maintain detailed records of customer interactions and complaints. (4) **Data governance** -- increased requirements for data quality, lineage, and reporting. (5) **Resolution planning** -- 'living wills' require banks to maintain detailed, accessible records of their operations. For GenAI, Dodd-Frank means any AI system used in trading, lending, or customer service must be auditable and explainable."

### Q10: What is PCI DSS and why does it matter for GenAI?

**Strong Answer**: "PCI DSS (Payment Card Industry Data Security Standard) is a set of security standards for organizations that handle credit card information. Key requirements: encrypt cardholder data, maintain secure networks, regularly monitor and test systems, implement access controls, maintain an information security policy. For GenAI applications that handle payment-related queries or process card data, PCI DSS compliance means: (1) Card numbers must never be sent to external LLM APIs. (2) Any system that processes, stores, or transmits card data must be PCI-compliant. (3) Access to card data must be logged and restricted. (4) Regular vulnerability scanning and penetration testing is required."

### Q11: What is Basel III and how does it affect banks?

**Strong Answer**: "Basel III is an international regulatory framework that sets minimum capital requirements, leverage ratios, and liquidity standards for banks. Key components: (1) **Capital requirements** -- banks must hold minimum capital as a percentage of risk-weighted assets (4.5% CET1 minimum). (2) **Leverage ratio** -- minimum 3% Tier 1 capital to total exposure. (3) **Liquidity Coverage Ratio (LCR)** -- enough high-quality liquid assets to survive a 30-day stress scenario. (4) **Net Stable Funding Ratio (NSFR)** -- stable funding for long-term assets. For technology teams, Basel III means extensive data collection, risk modeling, and reporting systems. GenAI could help with regulatory reporting automation but must be extremely accurate."

## Banking Operations

### Q12: How does a personal loan approval process work?

**Strong Answer**: "The typical process: (1) **Application** -- customer submits application with income, employment, credit information. (2) **Credit check** -- pull credit report and score from bureaus (Equifax, Experian, TransUnion). (3) **Underwriting** -- assess risk using credit score, debt-to-income ratio, employment history, collateral. Automated underwriting systems make instant decisions for standard cases. (4) **Verification** -- verify income (pay stubs, tax returns), employment, and identity. (5) **Approval/Denial** -- approve with terms (rate, amount, tenure) or deny with adverse action notice. (6) **Documentation** -- sign loan agreement, disclosures (Regulation Z). (7) **Funding** -- disburse funds to customer account. Total time: instant for pre-approved, 1-5 days for new applications. A GenAI assistant helping loan officers would need access to all these steps and the relevant policies at each stage."

### Q13: What is the difference between retail banking and corporate banking?

**Strong Answer**: "**Retail banking** serves individual consumers: checking/savings accounts, personal loans, mortgages, credit cards. High volume, low transaction value, standardized products. **Corporate banking** serves businesses: commercial loans, treasury services, trade finance, cash management, credit lines. Lower volume, high transaction value, customized solutions. The technology systems are typically separate: retail uses core banking systems optimized for high-volume consumer transactions, while corporate uses more flexible systems that handle complex credit structures and relationship-based pricing. For GenAI, this means separate knowledge bases, separate access controls, and different user personas for each banking line."

## Scenario-Based Banking Questions

### Q14: A customer reports an unauthorized transaction on their account. What regulations apply?

**Strong Answer**: "Multiple regulations apply: (1) **Regulation E** (Electronic Fund Transfers) -- if it's a debit card or electronic transfer. The customer's liability is limited to $50 if reported within 2 business days, $500 within 60 days, and potentially unlimited after 60 days. The bank must investigate within 10 business days (20 days for new accounts). (2) **Regulation Z** -- if it's a credit card. Maximum liability is $50, and the bank must resolve within 2 billing cycles (max 90 days). (3) **BSA/AML** -- if the unauthorized transaction suggests account takeover or money laundering, file a SAR. (4) **State laws** -- may provide additional consumer protections. The bank must also follow its own fraud investigation procedures and keep detailed audit records."

### Q15: How would you explain the concept of 'risk-weighted assets' to a non-technical person?

**Strong Answer**: "Think of it this way: a bank lends money to different types of borrowers, and some are riskier than others. A mortgage loan backed by a house is relatively safe -- if the borrower doesn't pay, the bank can sell the house. A credit card loan is riskier -- there's no collateral. Risk-weighted assets assign a 'risk weight' to each type of loan: mortgages might be weighted at 50% (only half the loan amount counts as risk), while credit cards are weighted at 100%. The total of all these weighted amounts is the bank's risk-weighted assets. Regulators then require the bank to hold a certain percentage of capital against this number -- more capital for riskier portfolios. It's like an insurance policy: the riskier your portfolio, the more safety net you need."

## Quick-Fire Banking Questions

### Q16: What is the Federal Reserve's role in banking?
**Answer**: "The Fed is the central bank of the US. It sets monetary policy (interest rates), regulates banks, provides financial services (check clearing, wire transfers), and acts as lender of last resort."

### Q17: What is a SAR (Suspicious Activity Report)?
**Answer**: "A report filed with FinCEN when a bank detects potentially suspicious transactions that might indicate money laundering, fraud, or other financial crimes. Must be filed within 30 days of detection."

### Q18: What is the difference between a debit card and a credit card?
**Answer**: "Debit card: money is immediately deducted from the customer's checking account. Credit card: the bank lends money for the purchase, and the customer repays later with potential interest."

### Q19: What is LIBOR and why is it being phased out?
**Answer**: "LIBOR was the benchmark interest rate for short-term interbank loans. Being phased out due to manipulation scandals. Replaced by SOFR (Secured Overnight Financing Rate) in the US."

### Q20: What is a bank's 'cost of funds'?
**Answer**: "The interest rate a bank pays to acquire the money it lends. For deposits, it's the interest paid to depositors. A lower cost of funds means higher net interest margin."

### Q21: What is net interest margin?
**Answer**: "The difference between interest income from loans and interest paid on deposits, divided by earning assets. It's a key profitability metric for banks."

### Q22: What is a correspondent bank?
**Answer**: "A bank that provides services on behalf of another bank, typically used for cross-border transactions, currency exchange, and accessing foreign markets."

### Q23: What is the FDIC and what does it insure?
**Answer**: "Federal Deposit Insurance Corporation. Insures deposits up to $250,000 per depositor per bank. Protects customers if the bank fails."

### Q24: What is a bank run?
**Answer**: "When many customers withdraw deposits simultaneously due to fear of bank insolvency. Can become a self-fulfilling prophecy. Prevented by FDIC insurance and Fed liquidity facilities."

### Q25: What is the difference between a commercial bank and an investment bank?
**Answer**: "Commercial banks accept deposits and make loans (consumer and business). Investment banks help companies raise capital (IPOs, bond issuance), provide M&A advisory, and facilitate trading."
