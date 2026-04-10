# Mock Interview Scripts

## How to Use These Scripts

1. **Pair with a partner**: One plays interviewer, one plays candidate
2. **Follow the time boxes**: Real interviews have strict time limits
3. **Don't skip the feedback**: 10 minutes of feedback is worth more than 30 minutes of practice
4. **Record yourself**: Review the recording for improvement areas
5. **Rotate roles**: Both people should practice being candidate and interviewer

---

## Mock Interview 1: Backend + GenAI (45 minutes)

### Introduction (2 min)
**Interviewer**: "Hi, I'm [name]. Today we'll cover backend fundamentals and GenAI concepts. I'll ask a mix of conceptual and scenario-based questions. Feel free to think out loud and ask clarifying questions. Let's start."

### Backend Fundamentals (15 min)

**Q1** (5 min): "Explain how you would design an idempotent payment API. What happens if the client retries due to a network timeout?"

*What to listen for*: Idempotency keys, database transactions, double-entry bookkeeping, retry handling.

**Q2** (5 min): "What is the N+1 query problem? Give a concrete example and show how you'd fix it."

*What to listen for*: Clear example, ORM eager loading or batch loading, understanding of why it happens.

**Q3** (5 min): "How would you debug a production memory leak in a Python service?"

*What to listen for*: Systematic approach, tools (tracemalloc, heap snapshots), common Python leak patterns, production safety measures.

### GenAI Concepts (15 min)

**Q4** (5 min): "Walk me through how a RAG system processes a user query from start to finish."

*What to listen for*: Complete pipeline (query -> embed -> retrieve -> re-rank -> assemble context -> generate -> validate), mention of each component's role.

**Q5** (5 min): "Your RAG system's retrieval precision dropped from 75% to 50%. How do you debug?"

*What to listen for*: Check recent changes, check embedding model consistency, check document freshness, run golden dataset, check vector DB health.

**Q6** (5 min): "How do you reduce hallucination in a banking RAG system?"

*What to listen for*: Multiple layers (system prompt, temperature=0, citation enforcement, retrieval quality gate, post-generation verification, human review).

### Scenario (10 min)

**Q7**: "Design a system that lets bank employees search across 10,000 policy documents. Users should be able to ask questions in natural language and get answers with citations."

*What to listen for*: Requirements clarification, hybrid search, access control, caching, LLM choice, audit logging, incremental indexing.

### Feedback (3 min)
**Interviewer**: Provide specific feedback on strengths and areas for improvement.

---

## Mock Interview 2: System Design (45 minutes)

### Introduction (2 min)
**Interviewer**: "Today we'll do a system design exercise. I'll give you a problem statement. Spend the first few minutes clarifying requirements, then walk me through your design. I'll ask follow-up questions along the way."

### Problem: Design an AI Model Gateway (40 min)

**Problem statement**: "Design a centralized gateway that manages access to multiple LLM providers (OpenAI, Anthropic, self-hosted) for all AI applications within the bank."

**Phase 1 - Requirements (5 min)**: Candidate should ask about scale, users, latency, cost constraints, security requirements.

**Phase 2 - High-level design (10 min)**: Candidate draws architecture with API gateway, model router, cache, usage tracker, fallback handler.

**Phase 3 - Deep dive (15 min)**: Interviewer probes:
- "How does the model router decide which model to use?"
- "How do you implement semantic caching?"
- "What happens when a provider goes down?"
- "How do you track and enforce per-team budgets?"
- "How do you prevent PII from being sent to external APIs?"

**Phase 4 - Tradeoffs (5 min)**: "What alternatives did you consider for the routing logic? Why did you choose Redis for caching?"

**Phase 5 - Scaling (5 min)**: "How does this system handle 10x growth in usage?"

### Feedback (3 min)

---

## Mock Interview 3: Behavioral + Leadership (30 minutes)

### Introduction (2 min)
**Interviewer**: "I'll ask behavioral questions about your past experiences. Use the STAR framework (Situation, Task, Action, Result). I'll ask follow-up questions."

### Questions (25 min)

**Q1** (5 min): "Tell me about a time you disagreed with a technical decision."
*Follow-up*: "How did you know your approach was better?" "What if the other person was right?"

**Q2** (5 min): "Tell me about a production failure you caused or were involved in."
*Follow-up*: "What did you change to prevent it from happening again?"

**Q3** (5 min): "Tell me about a time you had to explain a complex technical concept to a non-technical audience."
*Follow-up*: "How did you know they understood?"

**Q4** (5 min): "Tell me about a project that didn't go as planned."
*Follow-up*: "What would you do differently?"

**Q5** (5 min): "Tell me about a time you led without formal authority."
*Follow-up*: "How did you get buy-in?"

### Feedback (3 min)

---

## Mock Interview 4: Full Technical Loop Simulation (60 minutes)

### Round 1: Coding (15 min)
**Problem**: "Implement a token bucket rate limiter with configurable rate and capacity. Include tests."

### Round 2: System Design (20 min)
**Problem**: "Design a RAG-based policy search system for bank employees."

### Round 3: GenAI Deep Dive (15 min)
- "How do you choose chunk size?"
- "When would you skip re-ranking?"
- "How do you handle multi-language documents?"
- "What metrics do you track in production?"

### Round 4: Behavioral (10 min)
- "Tell me about your most impactful technical contribution."
- "How do you handle competing priorities?"

### Feedback (5 min)

---

## Interviewer Checklist

After each mock interview, rate the candidate on:

| Criterion | 1-5 | Notes |
|---|---|---|
| Clarifies requirements before diving in | | |
| Structures answers logically | | |
| Provides specific examples and numbers | | |
| Discusses tradeoffs without being prompted | | |
| Handles follow-up questions well | | |
| Admits uncertainty honestly | | |
| Banking context awareness | | |
| Communication clarity | | |
| Technical depth | | |
| Problem-solving approach | | |

## Candidate Self-Assessment

After each practice interview, ask yourself:
- Did I ask clarifying questions?
- Did I structure my answers?
- Did I include specific numbers and examples?
- Did I discuss tradeoffs?
- Did I handle follow-up questions well?
- Was I honest about what I don't know?
- What are my 2 biggest improvement areas?
