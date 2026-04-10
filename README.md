# Regulated Enterprise AI Engineering Handbook

[![License: CC BY-SA 4.0](https://img.shields.io/badge/License-CC%20BY--SA%204.0-lightgrey.svg)](LICENSE)
[![Docs Status](https://img.shields.io/badge/docs-558%20files-green)](README.md)
[![Sections](https://img.shields.io/badge/sections-29-blue)](README.md)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

> A comprehensive engineering handbook for building secure, compliant, production-grade AI systems in regulated environments.

---

## Overview

This is a **production-grade engineering handbook** for engineers building enterprise AI systems in **highly regulated environments**: global banks, fintechs, healthcare companies, insurers, and government organizations.

It covers the complete spectrum: backend engineering, frontend architecture, platform engineering, DevOps, security, compliance, GenAI systems, observability, testing, engineering culture, and technical leadership. All through the lens of **shipping reliable, auditable, safe software at scale**.

This is not a theoretical textbook. Every guide contains real banking examples, production-grade code samples, incident stories, interview preparation, and checklists you can use on Monday morning.

### Learning Path

```
Foundation              Build                   Platform                Operate                Grow
-------------------     --------------------    --------------------    --------------------   --------------------
Engineering Philosophy  Backend Engineering     GenAI Platforms         Observability          Engineering Culture
Banking Domain          Frontend Engineering    RAG and Search          Testing and Quality    Leadership
Regulations             Security Engineering    Kubernetes/OpenShift    Incident Management    Interview Prep
                        Databases               CI/CD and DevOps        Architecture           Exercises
                        Data Engineering        Infrastructure                                 Templates
                                                                                                 Glossary
```

## Who This Is For

| Role | How to Use This Handbook |
|------|------------------------|
| **Senior Engineers** building AI platforms | Deep-dive into `genai-platforms/`, `rag-and-search/`, `architecture/`, `system-design/` |
| **Staff/Principal Engineers** preparing for bank interviews | Start with `interview-prep/`, `system-design/`, `architecture/`, `real-world-case-studies/` |
| **Mid-Level Engineers** leveling up to senior | Follow the [30/60/90 day plan](#306090-day-study-plan) through `backend-engineering/`, `security/`, `kubernetes-openshift/` |
| **Engineering Managers** leading AI teams | Read `engineering-culture/`, `leadership-and-collaboration/`, `incident-management/` |
| **Security Engineers** reviewing AI systems | Focus on `security/`, `regulations-and-compliance/`, `genai-platforms/ai-safety.md` |
| **Platform Engineers** building internal platforms | Study `kubernetes-openshift/`, `cicd-devops/`, `infrastructure/`, `architecture/` |
| **Anyone** building RAG, agents, or enterprise AI | Start with `genai-platforms/`, `rag-and-search/`, `skills/` |

## Quick Navigation

<details>
<summary><b>Core Engineering</b></summary>

| Section | Files | Key Topics |
|---------|-------|-----------|
| [Backend Engineering](backend-engineering/) | 36+ | Python/FastAPI, Go, Java/Spring, TypeScript/NestJS, API design, resilience patterns |
| [Frontend Engineering](frontend-engineering/) | 19 | React, Next.js, streaming AI responses, citations, safe content rendering, accessibility |
| [Security Engineering](security/) | 24 | OWASP, OAuth2/OIDC, prompt injection, jailbreak defense, supply chain security |
| [Databases](databases/) | 23 | Postgres, pgvector, Redis, replication, HA, multi-tenant patterns |
| [Data Engineering](data-engineering/) | 26 | SQL, Kafka, Spark, CDC, data quality, embedding pipelines |
| [Testing and Quality](testing-and-quality/) | 22 | Unit/integration/E2E, chaos engineering, LLM evaluation, golden datasets |

</details>

<details>
<summary><b>AI and GenAI Systems</b></summary>

| Section | Files | Key Topics |
|---------|-------|-----------|
| [GenAI Platforms](genai-platforms/) | 34+ | LLM fundamentals, agents, tool calling, multi-model architecture, safety, 6 banking use cases |
| [RAG and Search](rag-and-search/) | 24 | Chunking, embeddings, retrieval, reranking, pgvector, evaluation, production incidents |
| [Architecture](architecture/) | 23 | ADRs, design patterns, 3 full architecture examples with Mermaid diagrams |
| [System Design](system-design/) | 15 | 14 complete system designs with requirements, tradeoffs, diagrams, interview Q&A |

</details>

<details>
<summary><b>Platform and Operations</b></summary>

| Section | Files | Key Topics |
|---------|-------|-----------|
| [Kubernetes and OpenShift](kubernetes-openshift/) | 23 | Pods, deployments, SCCs, operators, GPU workloads, blue/green, canary |
| [CI/CD and DevOps](cicd-devops/) | 21 | GitHub Actions, Tekton, Harness, Terraform, change management, progressive delivery |
| [Observability](observability/) | 22 | OpenTelemetry, SLOs, GenAI observability, token usage, hallucination monitoring |
| [Infrastructure](infrastructure/) | 13 | Cloud, networking, compute, storage, platform engineering, golden paths |
| [Incident Management](incident-management/) | 18 | Incident command, communication, GenAI-specific playbooks, banking obligations |

</details>

<details>
<summary><b>Domain Knowledge and Culture</b></summary>

| Section | Files | Key Topics |
|---------|-------|-----------|
| [Banking Domain](banking-domain/) | 13 | Banking 101, retail/investment/corporate banking, payments, AML, KYC, lending |
| [Regulations and Compliance](regulations-and-compliance/) | 21 | GDPR, PCI-DSS, SOC2, ISO 27001, BCBS 239, SR 11-7, AI governance, EU AI Act |
| [Engineering Philosophy](engineering-philosophy/) | 11 | Bias for action, pragmatism vs excellence, craftsmanship, collaboration |
| [Engineering Culture](engineering-culture/) | 16 | Code reviews, mentorship, pair programming, psychological safety |
| [Leadership](leadership-and-collaboration/) | 15 | Influence without authority, stakeholder management, executive communication |

</details>

<details>
<summary><b>Interview Prep and Practice</b></summary>

| Section | Files | Key Topics |
|---------|-------|-----------|
| [Interview Prep](interview-prep/) | 18 | 300+ questions across 10 categories, STAR stories, coding exercises |
| [Exercises](exercises/) | 22 | Hands-on exercises with solutions, hints, and extensions |
| [Templates](templates/) | 17 | Design docs, RFCs, ADRs, postmortems, runbooks, release plans |
| [Real-World Case Studies](real-world-case-studies/) | 10 | 10 detailed production incident stories with timelines and lessons |
| [Glossary](glossary/) | 11 | 250+ terms across banking, security, AI, K8s, backend, frontend, DevOps |
| [Onboarding](onboarding/) | 9 | 30/60/90 day plans, platform understanding, regulated environments |
| [Agents](agents/) | 13 | AI-assisted code review, architecture review, interview coaching |
| [Skills](skills/) | 18 | Reusable checklists: API design, threat modeling, prompt engineering, RAG evaluation |

</details>

## 30/60/90 Day Study Plan

<details>
<summary><b>First 30 Days: Build Foundation</b></summary>

| Week | Focus | Deliverables |
|------|-------|-------------|
| 1 | [Onboarding](onboarding/), [Philosophy](engineering-philosophy/), [Glossary](glossary/) | Read all onboarding docs, understand banking 101 |
| 2 | [Banking Domain](banking-domain/), [Regulations](regulations-and-compliance/) | Complete Banking 101, understand GDPR, PCI-DSS basics |
| 3 | [Backend Engineering](backend-engineering/) (Python + Go) | Build a FastAPI service, understand async patterns |
| 4 | [Security Fundamentals](security/) | Complete OWASP Top 10, threat modeling exercise |

**Checkpoint:** You should be able to explain why banking systems differ from typical SaaS, and describe the security implications of a GenAI assistant accessing internal documents.
</details>

<details>
<summary><b>Days 31-60: Deep Technical Mastery</b></summary>

| Week | Focus | Deliverables |
|------|-------|-------------|
| 5-6 | [GenAI Platforms](genai-platforms/) + [RAG](rag-and-search/) | Build a RAG pipeline with pgvector, implement guardrails |
| 7 | [Kubernetes/OpenShift](kubernetes-openshift/) | Deploy services, understand Routes, SCCs, Operators |
| 8 | [CI/CD](cicd-devops/) + [Observability](observability/) | Design a Tekton pipeline, set up OpenTelemetry |

**Checkpoint:** You should be able to design a secure RAG-based internal assistant, explain the deployment pipeline, and describe how you would monitor it in production.
</details>

<details>
<summary><b>Days 61-90: Architecture, Leadership, Interview Readiness</b></summary>

| Week | Focus | Deliverables |
|------|-------|-------------|
| 9-10 | [System Design](system-design/) + [Architecture](architecture/) | Complete 5+ system design exercises with Mermaid diagrams |
| 11 | [Leadership](leadership-and-collaboration/) + [Culture](engineering-culture/) | Practice stakeholder communication, write an executive summary |
| 12 | [Interview Prep](interview-prep/) + [Exercises](exercises/) | Complete all coding exercises, practice mock interviews |

**Checkpoint:** You should confidently handle backend, frontend, system design, and behavioral interviews. You should have strong opinions on architecture tradeoffs, backed by examples.
</details>

## Core Principles

### Engineering Principles
- **Systems thinking over feature thinking:** Every decision ripples through the entire system
- **Documentation is engineering:** If it is not documented, it did not happen
- **Automation over toil:** If you do something twice, automate it
- **Error budgets drive decisions:** Data-driven shipping, not gut-feel
- **Production first:** Design systems assuming they will fail in production

### Security Principles
- **Trust nothing:** Every input is untrusted, every request must be authenticated and authorized
- **Defense in depth:** No single control is perfect, layer your defenses
- **Least privilege:** No one gets more access than they need
- **Fail securely:** Errors never leak sensitive information or system internals
- **Shift left:** Security checklists in PR templates, automated scanning in CI

### AI Safety Principles
- **Safety and reliability over raw capability:** A less capable model that is reliable and safe is better
- **Multi-model by default:** Never depend on a single model provider
- **Prompts are code:** Versioned, tested, reviewed, managed like any other code
- **Measure, do not guess:** Golden datasets, regular evaluation, hallucination tracking
- **Human oversight for high-stakes:** Human-in-the-loop for decisions that matter

### Compliance Principles
- **Regulation-driven, not feature-driven:** Every requirement traced to a regulation
- **Evidence-first:** Concrete, auditable, time-stamped evidence for every claim
- **Data minimization:** Collect and retain only what is necessary
- **Traceability:** Every control traceable: Regulation to Policy to Control to Implementation to Evidence
- **Always audit-ready:** Never scrambling, always prepared

### Testing and Reliability Principles
- **Quality is built in, not inspected in:** Testing is a culture, not a phase
- **AI quality is different:** Probabilistic behavior requires golden datasets and human review
- **SLOs define reliability:** User-centric, measurable, actionable
- **Blameless postmortems:** Incidents are system failures, not human failures
- **Action items drive change:** Every postmortem produces real, tracked improvements

## How to Use This Handbook

### For Interview Preparation
1. Start with [`interview-prep/`](interview-prep/) for 300+ questions across 10 categories
2. Study [`system-design/`](system-design/) for 14 complete architecture designs
3. Practice [`exercises/`](exercises/) with 22 hands-on challenges and solutions
4. Review [`security/`](security/) and [`regulations-and-compliance/`](regulations-and-compliance/) for domain-specific questions
5. Use [`agents/interview-coach-agent.md`](agents/interview-coach-agent.md) for mock interviews

### For On-the-Job Reference
1. Use [`skills/`](skills/) for reusable checklists when building or reviewing
2. Reference [`templates/`](templates/) for design docs, RFCs, ADRs, and runbooks
3. Consult [`architecture/`](architecture/) for decision record patterns
4. Read [`incident-management/`](incident-management/) for incident response playbooks
5. Use the relevant [`agents/`](agents/) for AI-assisted code reviews

### For Real-World AI Projects
1. Follow the theory, code, design, reality loop below
2. Study [`real-world-case-studies/`](real-world-case-studies/) for production failure stories
3. Apply [`genai-platforms/`](genai-platforms/) patterns for safe AI architecture
4. Use [`skills/genai-guardrails.md`](skills/genai-guardrails.md) for AI safety controls

## How to Combine Theory, Coding, System Design, and Real-World Scenarios

Every section in this handbook is interconnected. The most effective learning path follows a loop:

```
THEORY          Read the concept in the relevant doc
  |            (e.g., skills/secure-api-design.md)
  v
CODE            Build it in exercises/ with constraints
  |            (e.g., Implement rate limiting in FastAPI)
  v
DESIGN          Architect it in system-design/
  |            (e.g., Design an AI Gateway Platform)
  v
REALITY         Study what went wrong in
                real-world-case-studies/ and
                incident-management/
```

**Example flow:**
1. **Theory:** Read [`skills/secure-api-design.md`](skills/secure-api-design.md)
2. **Code:** Complete [`exercises/coding-exercise-02-api-endpoint.md`](exercises/coding-exercise-02-api-endpoint.md)
3. **Design:** Study [`system-design/ai-gateway-platform.md`](system-design/ai-gateway-platform.md)
4. **Reality:** Read [`real-world-case-studies/rag-data-exfiltration-incident.md`](real-world-case-studies/rag-data-exfiltration-incident.md)

## AI Agent Prompts

This handbook includes 13 specialized agent instruction files in [`agents/`](agents/) for AI-assisted engineering:

| Agent | Use For |
|-------|---------|
| [Principal Engineer](agents/principal-engineer-agent.md) | Architecture review, technical strategy, second-order analysis |
| [Staff Backend Engineer](agents/staff-backend-engineer-agent.md) | API design review, production readiness, performance |
| [Frontend Architect](agents/frontend-architect-agent.md) | React component review, accessibility, streaming AI UIs |
| [Security Reviewer](agents/security-reviewer-agent.md) | Threat modeling, vulnerability identification, secure coding |
| [Compliance Reviewer](agents/compliance-reviewer-agent.md) | Regulatory alignment, audit readiness, evidence packages |
| [SRE](agents/sre-agent.md) | SLO design, runbooks, incident response, capacity planning |
| [Platform Engineer](agents/platform-engineer-agent.md) | Kubernetes, CI/CD, golden paths, developer experience |
| [GenAI Architect](agents/genai-architect-agent.md) | LLM systems, RAG design, prompt engineering, AI safety |
| [Code Review](agents/code-review-agent.md) | Code quality, best practices, patterns, consistency |
| [Testing](agents/testing-agent.md) | Test strategy, LLM evaluation, golden datasets |
| [Incident Response](agents/incident-response-agent.md) | Triage, debugging, blameless postmortems |
| [Engineering Manager](agents/engineering-manager-agent.md) | Team dynamics, delivery, stakeholder communication |
| [Interview Coach](agents/interview-coach-agent.md) | Mock interviews, STAR story practice, feedback |

**Example usage:** "Act as the security-reviewer-agent defined in `agents/security-reviewer-agent.md`. Review the following API endpoint for security issues: [your code]"

## Common Anti-Patterns to Avoid

<details>
<summary><b>Click for common engineering anti-patterns</b></summary>

| Anti-Pattern | The Problem | The Fix |
|-------------|-------------|---------|
| **Gold-plating** | Building for 10x scale before proving 1x | Build for today with a clear scaling path |
| **Analysis paralysis** | Weeks of evaluation when a 2-week spike would decide | Time-box research, build spikes, decide with evidence |
| **Security as gate** | Security review at end of development | Shift left: checklists in PR templates, automated scanning in CI |
| **Compliance as afterthought** | Asking "is this compliant?" after building | Involve compliance during design phase |
| **Single model dependency** | All AI calls to one provider, no fallback | Multi-model architecture with automatic failover |
| **Prompt spaghetti** | Prompts in code, duplicated, never versioned | Centralized prompt management with versioning and rollout controls |
| **Ivory tower architecture** | Elaborate diagrams no team can implement | Ground architecture in team capability and delivery timeline |
| **Silent failures** | Swallowing exceptions, returning empty results | Fail loudly with appropriate HTTP status codes |
| **Alerting on symptoms** | "Latency is high" without telling you why | Alert on causes (model down, DB slow) with latency as secondary |
| **Blaming individuals** | "The engineer took too long to respond" | Systemic fixes, not individual blame |

</details>

## Frequently Asked Questions

<details>
<summary><b>Is this handbook specific to one bank or technology?</b></summary>

No. The patterns, principles, and examples are applicable across global banks, fintechs, healthcare companies, insurers, and government organizations building AI systems. Technology examples use Python, TypeScript, Go, Java, React, Next.js, Postgres, Kubernetes, OpenShift, Tekton, and Harness, but the principles apply regardless of stack.
</details>

<details>
<summary><b>Do I need banking experience to use this handbook?</b></summary>

No. The [`banking-domain/`](banking-domain/) section teaches banking fundamentals from scratch. You will learn how banks work, how money moves, why systems are complex, and how GenAI can support each area, all written for engineers.
</details>

<details>
<summary><b>Is this useful for non-banking regulated environments?</b></summary>

Yes. Healthcare (HIPAA), government (FedRAMP), insurance, and other regulated industries face similar challenges: audit requirements, data protection, compliance reviews, and regulated change management. The patterns transfer directly.
</details>

<details>
<summary><b>How current is the AI/GenAI content?</b></summary>

The handbook covers foundational patterns that are model-agnostic: RAG architectures, prompt management, guardrails, evaluation, multi-model routing, safety controls, and observability. These patterns remain relevant as individual models evolve.
</details>

<details>
<summary><b>Can I use this to prepare for FAANG interviews?</b></summary>

Yes. The [`interview-prep/`](interview-prep/) section includes 300+ questions, 22 exercises, 14 system designs, and STAR stories. The banking-specific questions are a bonus for specialized roles, but the technical depth applies to any senior engineering interview.
</details>

<details>
<summary><b>Is this repository open-source?</b></summary>

Yes. It is licensed under [CC BY-SA 4.0](LICENSE). You are free to use, adapt, and contribute. See [CONTRIBUTING.md](CONTRIBUTING.md) for how to contribute.
</details>

## Contributing

Contributions are welcome. This handbook grows through community input.

- Report issues: [Open an issue](https://github.com/USERNAME/regulated-enterprise-ai-engineering-handbook/issues)
- Contribute content: Read [CONTRIBUTING.md](CONTRIBUTING.md)
- Security concerns: See [SECURITY.md](SECURITY.md)
- Ask questions: See [SUPPORT.md](SUPPORT.md)
- View roadmap: See [ROADMAP.md](ROADMAP.md)

Before contributing, please read:
1. [CONTRIBUTING.md](CONTRIBUTING.md) - How to contribute
2. [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) - Community standards
3. [SECURITY.md](SECURITY.md) - Reporting security concerns

## Recommended Tools

| Category | Tools |
|----------|-------|
| **LLM APIs** | OpenAI, Anthropic Claude, Google Gemini |
| **Open-Source Models** | Llama 3, Mistral, Mixtral, Gemma |
| **Frameworks** | LangChain, LlamaIndex, Haystack |
| **Vector Databases** | pgvector, Pinecone, Milvus, Weaviate, Qdrant |
| **API Frameworks** | FastAPI (Python), NestJS (TypeScript), Gin (Go), Spring Boot (Java) |
| **Infrastructure** | Kubernetes, Red Hat OpenShift, Docker, Helm |
| **CI/CD** | GitHub Actions, Tekton, Harness, ArgoCD |
| **Observability** | OpenTelemetry, Prometheus, Grafana, Jaeger, Sentry |
| **Databases** | PostgreSQL, Redis, Elasticsearch |
| **Security** | HashiCorp Vault, Trivy, OPA/Gatekeeper, Snyk |

## Recommended Reading

- *Designing Data-Intensive Applications* - Martin Kleppmann
- *Site Reliability Engineering* - Google SRE Team
- *Accelerate* - Nicole Forsgren, Jez Humble, Gene Kim
- *The Staff Engineer's Path* - Tanya Reilly
- *Building Secure and Reliable Systems* - Google
- *Fundamentals of Software Architecture* - Mark Richards, Neal Ford
- *AI Engineering* - Chip Huyen
- *Designing Machine Learning Systems* - Chip Huyen
- *A Philosophy of Software Design* - John Ousterhout
- *Team Topologies* - Matthew Skelton, Manuel Pais

## Suggested GitHub Topics

`ai-engineering` `genai` `rag` `enterprise-ai` `regulated-ai` `ai-safety` `ai-governance` `banking-technology` `fintech` `secure-coding` `kubernetes` `openshift` `python` `typescript` `fastapi` `react` `system-design` `engineering-handbook` `interview-prep` `compliance` `gdpr` `model-governance` `prompt-engineering` `ai-security` `platform-engineering`

## Suggested Repository Description

Engineering handbook for building secure, compliant, production-grade AI systems in regulated environments. Covers backend, frontend, platform engineering, security, compliance, GenAI, Kubernetes, CI/CD, observability, system design, and engineering leadership.

## License

This handbook is licensed under the [Creative Commons Attribution-ShareAlike 4.0 International License](LICENSE).

## Acknowledgements

This handbook draws on engineering practices from organizations like Google, Netflix, Stripe, Amazon, and the broader open-source community, adapted for the unique challenges of building AI systems in regulated environments.

---

Start with [`onboarding/first-30-days.md`](onboarding/first-30-days.md) and let us build something that matters.
