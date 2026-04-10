# Cross-Functional Collaboration

> "In a bank, no engineering team ships in isolation. Your work touches security, compliance, product, design, legal, risk, and operations. Your ability to collaborate across these functions determines your impact."

## The Cross-Functional Reality

A GenAI feature in a bank doesn't go from "idea" to "production" through engineering alone:

```
Idea
  │
  ├─── Product: Defines requirements, user stories, acceptance criteria
  ├─── Design: User experience, interface design, accessibility
  ├─── Engineering: Builds the system
  ├─── Security: Reviews for vulnerabilities, threat models
  ├─── Compliance: Reviews for regulatory alignment
  ├─── Legal: Reviews for liability, data usage rights, AI governance
  ├─── Risk: Models potential impact of failure
  ├─── Operations: Runs the system in production
  └─── Training: Creates user education materials
```

Each function has its own priorities, timelines, and ways of working. Collaboration means making this work without anyone becoming a bottleneck.

## Working with Product Management

### What Product Owns

- **What** to build and **why**
- User requirements and acceptance criteria
- Prioritization and sequencing
- User communication and change management

### What Engineering Owns

- **How** to build it
- Technical feasibility and timeline estimates
- Architecture and implementation decisions
- Operational readiness

### Collaboration Best Practices

| Practice | Why | Example |
|----------|-----|---------|
| Involve product in technical trade-off discussions | Product can provide user impact context | "We can build this in 2 weeks with approach A or 4 weeks with approach B. B gives us 10x better latency." |
| Engineering participates in requirement grooming | Catch infeasible requirements early | "This requirement assumes real-time model responses, but our p99 is 3 seconds." |
| Shared definition of "done" | Avoids "it works but doesn't meet user needs" | "Done means tested, deployed, documented, AND users can actually accomplish their goal" |
| Regular sync on progress | Prevents surprise delays | Weekly 15-min check-in, not a formal status meeting |

### Banking Context: GenAI Product Requirements

Product requirements for GenAI features must include:

- **Accuracy requirements:** What is the acceptable error rate? (For a compliance assistant, the bar is much higher than for an IT helpdesk assistant.)
- **Safety requirements:** What content must be blocked or flagged?
- **Citation requirements:** Must responses include source references?
- **Fallback behavior:** What happens when the model can't answer?
- **User transparency:** Must users know they're talking to an AI?

## Working with Design

### When to Involve Design

- Any user-facing interface (chat UI, admin dashboard, configuration pages)
- Any change to existing user workflows
- Accessibility reviews
- Brand and communication tone

### Collaboration Best Practices

| Practice | Why |
|----------|-----|
| Involve design at the problem-definition stage, not after engineering has built something | Design shapes the problem space, not just the pixels |
| Share technical constraints early | "The LLM response time means we need a streaming UI" |
| Review designs before implementation begins | Catching issues in Figma is cheaper than in React |
| Engineers review design prototypes for technical feasibility | "This animation is nice but will add 200ms to TTFB" |

### GenAI-Specific Design Challenges

Designing GenAI user interfaces has unique challenges:

- **Latency perception:** LLM responses stream in over seconds. How does the UI handle this gracefully?
- **Uncertainty communication:** How does the UI convey that the AI might be wrong?
- **Citation display:** How are source documents shown without overwhelming the response?
- **Feedback collection:** How do users flag incorrect or harmful responses?
- **Context management:** How do users understand what context the AI has access to?

## Working with Security

### When to Involve Security

- Any new system exposed to users (internal or external)
- Any change to authentication or authorization
- Any system handling PII, customer data, or financial data
- Any integration with external APIs (including LLM APIs)
- Any new data storage or processing pipeline
- Any GenAI system that processes user prompts

### What Security Reviews

| Area | What They Look For |
|------|-------------------|
| Threat modeling | What can an attacker do with this system? |
| Authentication | Is access properly controlled? |
| Data protection | Is data encrypted at rest and in transit? |
| Input validation | Can user input be weaponized? |
| Dependency review | Are libraries up to date and free of known CVEs? |
| GenAI-specific | Prompt injection, data exfiltration, model manipulation |

### How to Make Security Reviews Smooth

1. **Engage early.** Don't wait until the week before launch.
2. **Provide a design doc.** Security can't review what they can't see.
3. **Self-assess first.** Use the security checklist (`skills/security-checklist.md`) before the formal review.
4. **Be receptive.** Security findings are not personal criticism.
5. **Address findings with timelines.** "We'll fix this in the next sprint" is better than "We'll get to it eventually."

### Common Security Findings for GenAI Systems

| Finding | Severity | Fix |
|---------|----------|-----|
| Prompt injection vulnerability | Critical | Implement input validation and injection detection |
| PII in prompts sent to external LLM API | High | Implement PII redaction before API call |
| No rate limiting on GenAI endpoint | Medium | Add rate limiting per user/IP |
| Model responses not content-filtered | High | Add output content filtering |
| Audit logs don't capture model version | Medium | Add model version to audit record |
| Vector DB accessible without auth | Critical | Implement document-level access control |

## Working with Compliance

### What Compliance Cares About

| Regulation | What It Means for GenAI |
|------------|------------------------|
| **GDPR** | Right to explanation for AI decisions, data minimization in prompts |
| **SOX** | Audit trails for financial AI systems, change control |
| **Basel III/IV** | Model risk management for AI used in capital calculations |
| **PCI-DSS** | Never send card data to LLM APIs |
| **AI Act (EU)** | Risk classification, transparency, human oversight |
| **SR 11-7 (Fed/OCC)** | Model validation for AI used in banking decisions |

### How to Work with Compliance

1. **Understand the regulatory landscape.** Don't ask compliance "what do I need to comply?" — come with an understanding of what regulations apply.
2. **Provide evidence, not assertions.** "Our system is compliant" is not evidence. "Here is our audit log schema, access control design, and data retention policy" is.
3. **Engage at design time, not after build.** Compliance issues are cheaper to fix in a design doc than in production.
4. **Respect the process.** Compliance reviews take time because they carry regulatory risk. Build this into your timeline.

### Compliance Documentation for GenAI

Every GenAI system in production needs:

- **Model risk assessment:** What decisions does the AI make? What's the impact of wrong decisions?
- **Data flow diagram:** Where does data enter, where is it processed, where is it stored, where does it leave?
- **Access control matrix:** Who can access what data and under what conditions?
- **Audit trail design:** What is logged, where, for how long, who can access it?
- **Human oversight design:** When and how does a human review AI outputs?

## Working with Legal

### When Legal Needs to Be Involved

- Contracts with external AI providers (OpenAI, Anthropic, etc.)
- Terms of service for AI-generated content
- Data usage agreements (can we use this data to train/improve our models?)
- Liability questions (what happens when the AI gives wrong advice?)
- Intellectual property (who owns AI-generated content?)

### How to Work with Legal

1. **Frame the question clearly.** Legal can't answer "Is this okay?" They can answer "Here are the risks and mitigations for approach X vs. approach Y."
2. **Provide technical context.** Assume legal doesn't know how RAG works. Explain it.
3. **Get it in writing.** Legal opinions should be documented for future reference.

## Working with Risk Management

### What Risk Management Does

Risk teams model the potential impact of system failures:

```
Risk Question: "What happens if the GenAI assistant gives wrong
compliance advice to a loan officer?"

Risk Analysis:
├── Likelihood: 5% (based on model evaluation)
├── Impact per occurrence: $50K-500K (regulatory fine per wrong decision)
├── Annual exposure: 10,000 compliance queries
├── Expected annual loss: 5% × $200K × 10,000 = $10M
└── Mitigation cost: $500K (human-in-the-loop review)
```

### How to Work with Risk

1. **Provide honest model performance data.** Don't oversell accuracy.
2. **Describe failure modes clearly.** What does "the AI gets it wrong" look like?
3. **Propose mitigations.** Risk wants to manage risk, not eliminate it.
4. **Understand the risk appetite.** Not all risk needs to be eliminated — some needs to be accepted and managed.

## Collaboration Frameworks

### The RACI Matrix

For cross-functional projects, define who is:

| Role | Meaning | Example |
|------|---------|---------|
| **Responsible** | Does the work | Engineering builds the feature |
| **Accountable** | Ultimately answerable | Engineering manager |
| **Consulted** | Provides input | Security, Compliance, Legal |
| **Informed** | Kept in the loop | Product, stakeholders |

### Cross-Functional Kickoff Template

```markdown
# Cross-Functional Kickoff: [Project Name]

## What We're Building
[1-2 sentence description]

## Team
| Function | Representative | Role |
|----------|---------------|------|
| Engineering | [Name] | Build |
| Product | [Name] | Requirements |
| Design | [Name] | UX |
| Security | [Name] | Review |
| Compliance | [Name] | Review |
| Risk | [Name] | Assessment |

## Timeline
| Milestone | Date | Dependencies |
|-----------|------|-------------|
| Design complete | [Date] | — |
| Security review | [Date] | Design approved |
| Compliance review | [Date] | Security approved |
| Development complete | [Date] | Reviews passed |
| Production launch | [Date] | All sign-offs |

## Communication
- Weekly sync: [Day, Time]
- Decision channel: [Slack/Confluence]
- Escalation path: [Who to contact for blockers]
```

## Cross-Functional Anti-Patterns

| Anti-Pattern | What It Looks Like | Fix |
|--------------|-------------------|-----|
| **Throw it over the wall** | Engineering finishes, then sends to security | Involve security from the start |
| **Compliance as a gate** | "Compliance won't let us" without understanding why | Understand the regulation, not just the compliance team's opinion |
| **Product dictates technical decisions** | PM specifies implementation details | PM specifies outcomes, engineering specifies implementation |
| **Engineering ignores UX** | "Users will figure it out" | Involve design early |
| **Siloed decision-making** | Each function decides independently | Joint decision-making for cross-functional decisions |

## Cross-References

- `engineering-culture/design-docs.md` — Design docs (cross-functional review input)
- `leadership-and-collaboration/managing-stakeholders.md` — Stakeholder management
- `leadership-and-collaboration/influencing-without-authority.md` — Influencing across functions
- `security/threat-modeling.md` — Threat modeling with security team
- `regulations-and-compliance/ai-governance.md` — AI governance requirements
