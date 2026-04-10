# How to Use These Exercises

## Purpose

These exercises provide hands-on practice with the concepts covered throughout the Banking GenAI Engineering Academy. Each exercise is designed to be realistic, banking-context-relevant, and progressively challenging.

## Exercise Structure

Every exercise follows this format:

```
1. Problem Statement
   -> Banking context and scenario
   -> What you need to build or solve

2. Constraints and Requirements
   -> Functional requirements
   -> Non-functional requirements (performance, security, compliance)

3. Expected Output
   -> What success looks like
   -> How to verify your solution

4. Hints (Progressive)
   -> Hint 1: Subtle nudge in the right direction
   -> Hint 2: More specific guidance
   -> Hint 3: Direct approach suggestion

5. Example Solution
   -> Working code with explanations
   -> Banking-specific considerations

6. Extensions
   -> Additional challenges for advanced learners

7. Interview Relevance
   -> How this relates to real interview questions
```

## How to Approach Exercises

### For Self-Study

1. **Read the problem statement carefully**. Understand the banking context.
2. **Attempt the exercise before looking at hints**. Struggle productively for at least 30 minutes.
3. **Use hints progressively**. Start with the subtlest hint. Only move to the next if you are still stuck.
4. **Compare your solution with the example**. Note differences in approach.
5. **Try extensions** if you have time and want a deeper challenge.

### For Interview Preparation

1. **Time yourself**. Most coding exercises should be completable in 45-60 minutes.
2. **Think out loud**. Practice explaining your approach as you code.
3. **Discuss tradeoffs**. Interviewers care about your reasoning, not just your code.
4. **Review the interview relevance notes** after completing the exercise.

### For Team Workshops

1. **Pair up**. Work in pairs as you would in pair programming.
2. **Timebox**: 60 minutes for coding, 30 minutes for discussion.
3. **Compare solutions**. Multiple correct approaches usually exist.
4. **Discuss banking implications**. How would this change in a non-banking context?

## Exercise Categories

### Coding Exercises (Python)

Core programming skills applied to banking GenAI context:

| # | Exercise | Skills Tested | Difficulty |
|---|----------|--------------|------------|
| 01 | Rate Limiter | Algorithms, concurrency | Medium |
| 02 | API Endpoint | FastAPI, validation, auth | Medium |
| 03 | RAG Pipeline | Vector search, embeddings, prompting | Hard |
| 04 | Prompt Injection Defense | Security, prompt engineering | Hard |
| 05 | Streaming Response | Async, SSE, real-time | Medium |

### SQL Exercises

Database skills for banking:

| # | Exercise | Skills Tested | Difficulty |
|---|----------|--------------|------------|
| 01 | Complex Queries | Joins, aggregations, window functions | Medium |
| 02 | Query Optimization | Indexes, EXPLAIN, performance | Hard |

### Architecture Exercises

System design for banking GenAI:

| # | Exercise | Skills Tested | Difficulty |
|---|----------|--------------|------------|
| 01 | Enterprise Chatbot Design | System design, scalability | Hard |
| 02 | Audit Logging System | Architecture, compliance | Medium |

### Security and Compliance

| # | Exercise | Skills Tested | Difficulty |
|---|----------|--------------|------------|
| 01 | Threat Modeling | Security analysis | Hard |
| 02 | Compliance Review | Regulatory knowledge | Medium |

### Incident Response

| # | Exercise | Skills Tested | Difficulty |
|---|----------|--------------|------------|
| 01 | Incident Response Simulation | Debugging, communication | Hard |
| 02 | High Latency Debug | Observability, profiling | Medium |
| 03 | Memory Leak Debug | Profiling, debugging | Hard |

### GenAI-Specific

| # | Exercise | Skills Tested | Difficulty |
|---|----------|--------------|------------|
| 01 | Prompt Engineering | Prompt design, evaluation | Medium |
| 02 | RAG Evaluation | Evaluation frameworks | Medium |
| 03 | Postmortem Writing | Incident analysis, writing | Medium |

### Infrastructure

| # | Exercise | Skills Tested | Difficulty |
|---|----------|--------------|------------|
| 01 | Kubernetes Debugging | K8s operations, debugging | Hard |
| 02 | CI/CD Troubleshooting | Pipeline debugging | Medium |
| 03 | Secure Coding | Security, code review | Hard |
| 04 | Design Review | Architecture critique | Medium |

## Difficulty Levels

```
┌─────────────────────────────────────────────────────────────┐
│  DIFFICULTY GUIDE                                            │
├──────────┬──────────────────────────────────────────────────┤
│  Easy    │  Straightforward implementation. Focus on        │
│          │  correctness and basic best practices.            │
│          │  Time: 20-30 minutes                              │
├──────────┼──────────────────────────────────────────────────┤
│  Medium  │  Requires understanding of multiple concepts.     │
│          │  May involve edge cases and error handling.       │
│          │  Time: 30-60 minutes                              │
├──────────┼──────────────────────────────────────────────────┤
│  Hard    │  Complex problem with multiple valid approaches.  │
│          │  Requires tradeoff analysis and design decisions. │
│          │  Time: 60-120 minutes                             │
└──────────┴──────────────────────────────────────────────────┘
```

## Prerequisites

Before attempting these exercises, you should be comfortable with:

- **Python**: Functions, classes, decorators, async/await
- **FastAPI**: Basic API endpoint creation
- **SQL**: SELECT, JOIN, GROUP BY, subqueries
- **Git**: Basic version control operations
- **Docker**: Running containers, docker-compose

For GenAI exercises:
- **LLM APIs**: Making API calls to OpenAI or similar
- **Embeddings**: Understanding of vector embeddings
- **RAG concepts**: Retrieval, augmentation, generation pipeline

## Getting Help

If you are stuck:

1. Review the relevant documentation in this academy
2. Use the progressive hints in each exercise
3. Search for specific technical questions online
4. Discuss with peers or mentors (for workshop settings)
5. Review the example solution and work backward

## Tracking Progress

Track your completion:

```
┌─────────────────────────────────────────────────────────────┐
│  EXERCISE PROGRESS                                           │
├─────────────────────────────┬───────┬───────┬───────────────┤
│  Exercise                   │ Done  │ Hints │  Notes        │
│                             │       │ Used  │               │
├─────────────────────────────┼───────┼───────┼───────────────┤
│ Coding 01: Rate Limiter    │ [ ]   │       │               │
│ Coding 02: API Endpoint    │ [ ]   │       │               │
│ Coding 03: RAG Pipeline    │ [ ]   │       │               │
│ ...                         │       │       │               │
└─────────────────────────────┴───────┴───────┴───────────────┘
```

Mark Done when you have a working solution. Note how many hints you used (0 = no hints, 3 = all hints).
