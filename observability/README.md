# Observability in Banking GenAI Engineering

## Philosophy

Observability is not a feature -- it is a foundational capability that enables everything else to function. In a banking environment where regulatory compliance, customer trust, and system reliability are non-negotiable, you cannot manage what you cannot observe.

Observability differs from monitoring. Monitoring tells you when something is wrong. Observability tells you **why** something is wrong. Monitoring answers the question "Is the system healthy?" Observability answers "Why is the system behaving this way?"

### The Three Pillars

Observability rests on three pillars, each providing a distinct lens into system behavior:

```
┌─────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY                             │
├──────────────────┬──────────────────┬───────────────────────┤
│    LOGS          │    METRICS       │    TRACES             │
│                  │                  │                       │
│  What happened?  │  How much/often? │  How did it flow?     │
│                  │                  │                       │
│  Discrete events │  Aggregated data │  Request lifecycle    │
│  with context    │  over time       │  across services      │
│                  │                  │                       │
│  "Error in auth" │  "500 err/s"     │  "API→Auth→DB (2s)"   │
└──────────────────┴──────────────────┴───────────────────────┘
```

**Logs** are discrete events with rich context. They capture what happened at a specific moment. In banking, logs serve dual purposes: operational debugging and regulatory audit trails. Every log entry must be structured, timestamped, and correlated with a request ID.

**Metrics** are numerical measurements aggregated over time. They answer quantitative questions: How many requests per second? What is the p99 latency? How much memory is the model server consuming? Metrics are cheap to store and excellent for alerting.

**Traces** follow a single request as it flows through distributed systems. They reveal the path, timing, and dependencies of each operation. In a GenAI banking application, a trace might flow: API Gateway -> Auth Service -> RAG Pipeline -> Vector DB -> LLM Provider -> Response Formatter -> Client.

### Why Banking Demands Higher Standards

Banking GenAI systems have observability requirements that exceed typical SaaS applications:

1. **Regulatory Audit Requirements**: Every AI decision affecting a customer must be traceable. Regulators require evidence of how a loan recommendation was generated, which data sources were consulted, and what model version was used.

2. **Customer Trust**: If a banking customer receives incorrect financial advice from an AI assistant, the impact goes beyond a bad user experience -- it can result in regulatory action and loss of banking license.

3. **Data Sensitivity**: Observability data itself must be protected. Logs must not contain PII, PCI data, or account numbers. Correlation IDs must not be guessable.

4. **Multi-Party Systems**: Banking GenAI often involves external model providers (OpenAI, Anthropic, custom models), vector databases, document processing pipelines, and core banking integrations. Tracing across organizational boundaries is essential.

5. **Cost Visibility**: LLM API calls are expensive. Token usage must be tracked per service, per team, per customer segment to manage costs.

## Observability Maturity Model

```
Level 0: No Observability          - Reacting to customer complaints
Level 1: Basic Monitoring          - CPU, memory, uptime dashboards
Level 2: Structured Logging        - JSON logs with correlation IDs
Level 3: Distributed Tracing       - Request flow across services
Level 4: SLO-Driven Operations     - Error budgets, burn rate alerting
Level 5: Business Observability    - Linking system health to business outcomes
Level 6: AI-Native Observability   - Model quality, drift, hallucination detection
```

Most banking teams should target Level 4 as a minimum. GenAI-specific teams need Level 6 capabilities.

## Table of Contents

- [Logging](logging.md) -- Structured logging practices
- [Structured Logs](structured-logs.md) -- JSON format and field conventions
- [Metrics](metrics.md) -- Metric types, naming conventions, labels
- [Tracing](tracing.md) -- Distributed tracing fundamentals
- [OpenTelemetry](opentelemetry.md) -- SDK setup and configuration
- [Dashboards](dashboards.md) -- Dashboard design and best practices
- [SLOs](slos.md) -- Service Level Objectives and error budgets
- [SLIs](slis.md) -- Service Level Indicator selection
- [Alerting](alerting.md) -- Alert design and routing
- [Error Budgets](error-budgets.md) -- Budget policy and consumption
- [Synthetic Monitoring](synthetic-monitoring.md) -- Proactive uptime checks
- [Frontend Observability](frontend-observability.md) -- RUM and Web Vitals
- [Backend Observability](backend-observability.md) -- Service-level observability
- [GenAI Observability](genai-observability.md) -- Model-specific observability
- [Prompt Logging](prompt-logging.md) -- Safe prompt logging practices
- [Model Latency](model-latency.md) -- Latency monitoring
- [Token Usage](token-usage.md) -- Token tracking and cost management
- [Hallucination Monitoring](hallucination-monitoring.md) -- Detecting AI hallucinations
- [User Feedback Analytics](user-feedback-analytics.md) -- Satisfaction tracking
- [Audit Dashboards](audit-dashboards.md) -- Compliance dashboards
- [Incident Debugging Playbooks](incident-debugging-playbooks.md) -- Debugging guides

## Banking Context Example

Consider a GenAI-powered mortgage advisory system:

```
Customer Query: "Can I afford a $450,000 home with my $95,000 salary?"

Observability chain:
1. Frontend captures Web Vitals (FCP, LCP, CLS)
2. API Gateway logs request with correlation-id: abc-123
3. Auth service metric: auth_duration_ms{user_tier=premium}
4. RAG pipeline traces document retrieval: 12 documents, 340ms
5. Vector DB query metric: query_latency_p99_ms
6. LLM call tracked: model=gpt-4, tokens_in=1200, tokens_out=450, cost=$0.036
7. Risk scoring service: risk_score=0.23 (low risk)
8. Response includes disclaimer (regulatory requirement logged)
9. Customer thumbs-down feedback captured and routed to review queue
10. Full trace archived for 7 years (regulatory retention)
```

Every step generates logs, metrics, and trace data that must be correlated, queryable, and retained according to banking regulations.

## Key Principles

1. **Observability is a Product Requirement**: It is not an afterthought. Design systems to be observable from day one.

2. **Structured Data Over Text**: All logs must be structured JSON. Text logs are impossible to query at scale.

3. **Correlation IDs Everywhere**: Every log, metric, and trace in a single request flow must share a correlation ID.

4. **PII Must Be Redacted**: Observability systems must never store customer names, account numbers, SSNs, or card numbers.

5. **SLOs Drive Decisions**: Without SLOs, you cannot prioritize reliability work. SLOs create objective criteria for launch decisions.

6. **Alert on Symptoms, Not Causes**: Alert when users are affected, not when CPU is high.

7. **Cost Is Observable**: Token usage, API costs, and infrastructure spend must be visible per service and team.
