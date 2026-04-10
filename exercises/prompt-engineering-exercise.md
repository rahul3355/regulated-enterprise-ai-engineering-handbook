# Prompt Engineering Exercise: Banking Assistant

> Solve real-world prompt engineering challenges for a banking GenAI assistant.

## Problem Statement

You are the prompt engineer for the bank's internal GenAI assistant. The assistant must provide accurate, safe, and helpful answers about banking policies, procedures, and systems. Solve the following prompt engineering challenges.

## Challenges

### Challenge 1: Accuracy — Preventing Hallucination

**Problem:** When the assistant doesn't know the answer, it sometimes makes one up.

```
User: "What is the capital requirement for derivatives trading under
       the new Basel III.5 framework?"

Bad response: "Under Basel III.5, the capital requirement for
               derivatives trading is 6.5% for cleared derivatives
               and 10% for uncleared derivatives."
               (Fabricated numbers — the assistant doesn't actually
               have this information in its context.)

Good response: "I don't have specific information about capital
                requirements for derivatives trading under Basel III.5
                in our policy documents. Please contact the Risk
                Management team for guidance on this topic."
```

**Task:** Write a system prompt that prevents hallucination while still being helpful when the information IS available.

### Challenge 2: Access Control — Preventing Data Leakage

**Problem:** The assistant has access to documents at different classification levels. It must not reveal Confidential or Restricted information to employees who don't have clearance.

```
System: The assistant receives retrieved documents as context,
        some of which may be at a higher classification than the
        user's clearance level.

User (clearance: Internal): "What are the investment algorithms
                             used by the proprietary trading desk?"

Retrieved context includes:
- PUBLIC: General trading policy overview
- INTERNAL: Trading hours and approval workflows
- CONFIDENTIAL: Investment algorithm descriptions (SHOULD NOT BE USED)

Bad response: (Uses CONFIDENTIAL document in response)
Good response: (Uses only PUBLIC and INTERNAL documents)
```

**Task:** Design the prompt template that enforces classification boundaries.

### Challenge 3: Response Format — Consistent Structure

**Problem:** The assistant's responses vary in format, making them hard to read and parse.

**Desired format:**
```
## Answer
[Direct answer to the question]

## Source
- [Document title] (v[version], effective [date])
- [Relevant excerpt]

## Confidence
[High/Medium/Low — based on document recency and relevance]

## Related
[If available: related policies the user might want to know about]

## Disclaimer
[If the answer involves compliance/regulatory information]
```

**Task:** Write the prompt template that produces this format consistently.

### Challenge 4: Multi-Turn Conversations

**Problem:** The assistant loses context between turns in a conversation.

```
Turn 1:
User: "What is the expense reporting deadline?"
Assistant: "Expenses must be submitted within 30 days..."

Turn 2:
User: "What about international expenses?"
Assistant: (Should understand this is a follow-up about expense
           reporting, not a general question about international things)

Turn 3:
User: "And what receipts do I need?"
Assistant: (Should understand "receipts" refers to expense receipts,
           not some other type of receipt)
```

**Task:** Design the conversation history format that maintains context without exceeding token limits.

### Challenge 5: Handling Ambiguity

**Problem:** User queries are often ambiguous.

```
Ambiguous query: "What's the policy on travel?"

This could mean:
- Domestic travel approval process
- International travel restrictions
- Travel expense reimbursement
- Travel insurance coverage
- Corporate card usage for travel
- Travel booking procedures

Bad response: Picks one interpretation and answers
Good response: Acknowledges ambiguity, asks for clarification,
               provides brief overview of travel-related policies
```

**Task:** Write the prompt handling logic for ambiguous queries.

## Example Solutions

### Solution 1: Anti-Hallucination System Prompt

```
You are a banking policy assistant. Your role is to help employees
find information in our internal policy documents.

CRITICAL RULES:
1. Answer ONLY using the provided context documents. Do not use
   any outside knowledge.
2. If the context does not contain sufficient information to answer
   the question, say: "I don't have specific information about [topic]
   in our policy documents. Please contact [relevant team] for
   guidance."
3. Never invent numbers, dates, percentages, or policy details.
4. If you are uncertain, say so. It is always better to direct the
   user to the right team than to provide potentially incorrect
   information.
5. Always cite which document your answer comes from.

Context documents:
{context}

User question: {query}
```

### Solution 2: Classification-Aware Prompt

```
You are a banking policy assistant. You will receive context documents
with classification labels.

CLASSIFICATION RULES:
- The user's clearance level is: {user_clearance}
- You may ONLY use documents classified at or below the user's
  clearance level.
- Documents classified above the user's clearance level must be
  IGNORED completely. Do not reference them, hint at them, or
  acknowledge their existence.
- If the available documents (at the user's clearance level) do not
  contain the answer, say so. Do not attempt to answer using
  restricted documents.

Available context (at or below {user_clearance} clearance):
{filtered_context}

User question: {query}
```

### Solution 3: Response Format Prompt

```
Answer the user's question using the provided context. Format your
response as follows:

## Answer
[Provide a direct, clear answer in 2-4 sentences. If you cannot
answer, explain why and direct the user to the right team.]

## Source
- [Document Title] (v[version], effective [date])
  [1-2 sentence relevant excerpt in quotes]

## Confidence
[Choose one:]
- High: The information is from a recent policy document and directly
  addresses the question.
- Medium: The information is from an older document or only partially
  addresses the question.
- Low: The information is inferred from general policy language or
  the documents only indirectly relate to the question.

## Related
[List 1-2 related policies the user might find helpful, or "No related
policies identified."]

## Disclaimer
[If this answer involves compliance, regulatory, or legal matters, add:
"This information is sourced from internal policy documents and is
provided for general guidance. For specific compliance decisions,
please consult the Compliance team directly."]
```

### Solution 4: Conversation History

```python
def build_prompt_with_history(
    query: str,
    history: list[dict],
    context: str,
    max_history_tokens: int = 500
):
    """Build prompt with conversation history, respecting token limits."""
    # Truncate history to fit within token budget
    total_tokens = 0
    truncated_history = []
    for turn in reversed(history):  # Most recent first
        turn_tokens = len(turn["user"]) // 4 + len(turn["assistant"]) // 4
        if total_tokens + turn_tokens > max_history_tokens:
            break
        truncated_history.insert(0, turn)
        total_tokens += turn_tokens

    # Build conversation section
    conversation = ""
    for turn in truncated_history:
        conversation += f"User: {turn['user']}\n"
        conversation += f"Assistant: {turn['assistant']}\n\n"

    prompt = f"""You are a banking policy assistant. You are in a
conversation with a user. Here is the conversation history:

{conversation}
The user's latest message is: {query}

Context documents:
{context}

Answer the user's latest question using the context and conversation
history. Maintain the response format."""

    return prompt
```

## Extensions

1. **Prompt versioning:** Design a prompt versioning system. How do you test that a new prompt version doesn't degrade response quality?

2. **Prompt A/B testing:** Design an experiment to compare two prompt templates using user satisfaction as the metric.

3. **Prompt injection defense:** Write system prompts that are resilient against injection attacks while remaining helpful.

4. **Multilingual prompts:** Adapt the prompts for 12 languages. How do you ensure consistent behavior across languages?

5. **Prompt evaluation pipeline:** Build an automated evaluation pipeline that scores prompt quality on accuracy, safety, helpfulness, and format compliance.

## Interview Relevance

Prompt engineering is tested in all GenAI roles:

| Skill | Why It Matters |
|-------|---------------|
| Prompt design | Core GenAI engineering skill |
| Hallucination prevention | Safety-critical in banking |
| Format control | Consistent, parseable outputs |
| Context management | Handling multi-turn, access control |
| Evaluation | Measuring prompt quality |

**Follow-up questions:**
- "How do you know if a prompt is good?"
- "What's the difference between zero-shot and few-shot prompting?"
- "How do you debug a prompt that produces bad outputs?"
- "What's chain-of-thought prompting and when would you use it in banking?"
