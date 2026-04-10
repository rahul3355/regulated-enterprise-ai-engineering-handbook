# Banking Terms Glossary

## AML (Anti-Money Laundering)

**Definition**: Regulations and procedures designed to prevent criminals from disguising illegally obtained funds as legitimate income.

**Banking Example**: The GenAI advisory system must flag conversations where a customer asks about structuring deposits to avoid reporting thresholds. Such conversations are logged and reported to the AML compliance team.

**Related Concepts**: KYC, SAR (Suspicious Activity Report), CTR (Currency Transaction Report), FinCEN

**Common Misunderstanding**: "AML only applies to large transactions." AML also applies to patterns of behavior, such as frequent small transactions that appear designed to avoid reporting thresholds.

**Interview Relevance**: MODERATE. Important for understanding banking context, rarely tested directly in engineering interviews.

---

## DTI (Debt-to-Income Ratio)

**Definition**: The percentage of a borrower's gross monthly income that goes toward paying debts. Calculated as: Total Monthly Debt Payments / Gross Monthly Income.

**Banking Example**: A customer with $95,000 annual income ($7,916/month) and monthly debts of $2,500 (mortgage, car loan, student loans) has a DTI of 31.6%. Most conventional mortgage lenders require DTI below 43%.

**Related Concepts**: LTV, Credit Score, Mortgage Eligibility, Underwriting

**Common Misunderstanding**: "DTI includes all living expenses." DTI only includes debt payments (loans, credit cards, mortgages), not living expenses like utilities, groceries, or insurance.

**Interview Relevance**: HIGH. Frequently relevant for mortgage-related system design and domain modeling questions.

---

## KYC (Know Your Customer)

**Definition**: The process of verifying the identity of clients and assessing their risk profiles before and during a business relationship.

**Banking Example**: Before a customer can use the GenAI advisory system for personalized advice, they must complete KYC verification: government-issued ID, proof of address, and risk tolerance assessment.

**Related Concepts**: AML, CDD (Customer Due Diligence), PEP (Politically Exposed Person), Identity Verification

**Common Misunderstanding**: "KYC is a one-time process." KYC is ongoing. Customer risk profiles are reassessed periodically and when significant changes occur.

**Interview Relevance**: MODERATE. Important context for authentication and authorization system design.

---

## LTV (Loan-to-Value Ratio)

**Definition**: The ratio of the loan amount to the appraised value of the underlying asset, expressed as a percentage.

**Banking Example**: For a $450,000 home with a $90,000 down payment, the loan amount is $360,000 and the LTV is 80%. An LTV above 80% typically requires Private Mortgage Insurance (PMI).

**Related Concepts**: DTI, PMI, Down Payment, Equity, Underwriting

**Common Misunderstanding**: "LTV is calculated at origination only." LTV changes over time as the borrower pays down the loan and the property value changes. Current LTV affects refinancing options.

**Interview Relevance**: HIGH. Commonly used in mortgage system calculations and eligibility logic.

---

## Basel III

**Definition**: International regulatory framework for bank capital adequacy, stress testing, and market liquidity risk, developed after the 2008 financial crisis.

**Banking Example**: The bank's GenAI platform must not introduce operational risk that would affect the bank's Basel III capital calculations. System outages in customer-facing services could be classified as operational risk events.

**Related Concepts**: Capital Adequacy Ratio, Tier 1 Capital, Stress Testing, Operational Risk

**Common Misunderstanding**: "Basel III applies only to investment banks." Basel III applies to all banks with significant operations, including retail and commercial banks.

**Interview Relevance**: LOW for engineers, but important context for understanding why banking regulations exist.

---

## Underwriting

**Definition**: The process by which a lender evaluates the risk of lending to a particular borrower and decides whether to approve the loan and on what terms.

**Banking Example**: The GenAI mortgage advisor does not underwrite loans. It provides information and guidance based on general underwriting criteria. The actual underwriting decision is made by the bank's underwriting system and human underwriters.

**Related Concepts**: DTI, LTV, Credit Score, Risk Assessment, Automated Underwriting

**Common Misunderstanding**: "AI can replace human underwriters entirely." In regulated banking, final underwriting decisions often require human review, especially for edge cases. AI can assist but not fully replace human judgment.

**Interview Relevance**: HIGH. Underwriting logic is a common domain modeling exercise in banking interviews.

---

## APR (Annual Percentage Rate)

**Definition**: The annual cost of borrowing, including interest and fees, expressed as a percentage. APR is typically higher than the nominal interest rate because it includes additional costs.

**Banking Example**: A mortgage with a 6.5% interest rate and $3,000 in origination fees on a $450,000 loan might have an APR of 6.72%. The GenAI advisor must clearly distinguish between interest rate and APR when providing information.

**Related Concepts**: Interest Rate, APY (Annual Percentage Yield), Origination Fee, Points

**Common Misunderstanding**: "APR and interest rate are the same." APR includes fees; the interest rate does not. APR gives a more accurate picture of total borrowing cost.

**Interview Relevance**: MODERATE. Important for financial calculation accuracy in advisory systems.

---

## PCI-DSS (Payment Card Industry Data Security Standard)

**Definition**: Security standards for organizations that handle cardholder information. Applies to all entities that store, process, or transmit credit card data.

**Banking Example**: If the GenAI platform processes credit card information (e.g., for fee payments), it must be PCI-DSS compliant. This includes encrypting card data, restricting access, and regular security audits.

**Related Concepts**: PII, Tokenization, Encryption, Security Scanning

**Common Misunderstanding**: "PCI-DSS only applies if you store card numbers." It also applies to processing and transmitting card numbers, even if you do not store them.

**Interview Relevance**: MODERATE. Important for security-related system design.

---

## Wire Transfer

**Definition**: An electronic transfer of funds across a network administered by banks or transfer agencies.

**Banking Example**: The GenAI advisor can explain wire transfer fees and processing times but cannot initiate transfers. Transfer initiation requires separate authentication and confirmation steps.

**Related Concepts**: ACH, SWIFT, Real-Time Payments, AML

**Common Misunderstanding**: "Wire transfers and ACH transfers are the same." Wire transfers are faster (same-day or next-day) but more expensive. ACH transfers take 1-3 business days but are typically free.

**Interview Relevance**: LOW. Context for understanding payment system differences.

---

## FDIC Insurance

**Definition**: Federal Deposit Insurance Corporation insurance that protects depositors against the loss of insured deposits if a bank fails.

**Banking Example**: The GenAI advisor can explain FDIC coverage limits ($250,000 per depositor, per insured bank) but should not provide advice on how to maximize coverage through complex structuring, which could be construed as legal advice.

**Related Concepts**: Deposit Insurance, NCUA (for credit unions), Systemic Risk

**Common Misunderstanding**: "All bank products are FDIC insured." Only deposit products (checking, savings, CDs) are insured. Investment products (stocks, bonds, mutual funds) offered through the bank are NOT insured.

**Interview Relevance**: LOW. Context for banking regulations.
