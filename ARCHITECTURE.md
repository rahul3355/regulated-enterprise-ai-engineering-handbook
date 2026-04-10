# Architecture

This document describes the **architectural design of this repository** — how content is organized, how sections relate to each other, and how to navigate and extend the handbook.

## Repository Architecture

```
regulated-enterprise-ai-engineering-handbook/
│
├── 📄 README.md                           ← Landing page, navigation, quick start
├── 📄 CONTRIBUTING.md                     ← How to contribute
├── 📄 CODE_OF_CONDUCT.md                  ← Community standards
├── 📄 SECURITY.md                         ← Security policy and reporting
├── 📄 SUPPORT.md                          ← How to get help
├── 📄 LICENSE                             ← CC BY-SA 4.0
├── 📄 ROADMAP.md                          ← Planned improvements
├── 📄 CHANGELOG.md                        ← Release history
├── 📄 ARCHITECTURE.md                     ← This file
│
├── 🤖 agents/                             ← AI agent role instructions (13 files)
├── 📋 skills/                             ← Reusable checklists (18 files)
├── 🚀 onboarding/                         ← New engineer ramp-up (9 files)
├── 🧠 engineering-philosophy/             ← Mindset and culture (11 files)
├── 🏦 banking-domain/                     ← Banking 101 for engineers (13 files)
├── 📜 regulations-and-compliance/         ← Regulatory guides (21 files)
├── 🔒 security/                           ← Security engineering (24 files)
├── ⚙️ backend-engineering/               ← Python, Go, Java, TypeScript (36+ files)
├── 🎨 frontend-engineering/              ← React, Next.js, enterprise UI (19 files)
├── 🤖 genai-platforms/                   ← LLM systems, RAG, agents (34+ files)
├── 🔍 rag-and-search/                     ← RAG pipeline deep-dive (24 files)
├── 📊 data-engineering/                   ← Pipelines and governance (26 files)
├── 🗄️ databases/                          ← Postgres, Redis, vector DBs (23 files)
├── ☁️ infrastructure/                     ← Cloud and platform (13 files)
├── ☸️ kubernetes-openshift/              ← K8s and OpenShift operations (23 files)
├── 🔄 cicd-devops/                        ← CI/CD and DevOps (21 files)
├── 📈 observability/                      ← Logs, metrics, traces, SLOs (22 files)
├── 🧪 testing-and-quality/               ← Testing strategies, LLM eval (22 files)
├── 🏗️ architecture/                       ← ADRs, patterns, examples (23 files)
├── 📐 system-design/                      ← Interview and real-world designs (15 files)
├── 📖 real-world-case-studies/            ← Production stories (10 files)
├── 🚨 incident-management/                ← Incident response (18 files)
├── 👥 engineering-culture/                ← Team practices (16 files)
├── 🤝 leadership-and-collaboration/       ← Influence, communication (15 files)
├── 💼 interview-prep/                     ← Questions, exercises, stories (18 files)
├── 🏋️ exercises/                          ← Hands-on practice (22 files)
├── 📝 templates/                          ← Design docs, RFCs, runbooks (17 files)
├── 📚 glossary/                           ← Terms across all domains (11 files)
│
└── .github/                               ← GitHub community files
    ├── ISSUE_TEMPLATE/                    ← Issue templates (5 types)
    ├── workflows/                         ← CI workflows (4 workflows)
    ├── PULL_REQUEST_TEMPLATE.md           ← PR template
    ├── CODEOWNERS                         ← Review routing
    └── dependabot.yml                     ← Dependency updates
```

## Content Layering

The handbook is structured in four learning layers:

```
┌─────────────────────────────────────────────────────────────────┐
│ Layer 4: Lead & Grow                                            │
│ architecture/ system-design/ engineering-culture/              │
│ leadership/ interview-prep/ exercises/ templates/              │
│                                                                 │
│ Purpose: System design skills, leadership, career growth       │
├─────────────────────────────────────────────────────────────────┤
│ Layer 3: Operate at Scale                                       │
│ genai-platforms/ rag-and-search/ kubernetes-openshift/         │
│ cicd-devops/ observability/ infrastructure/                    │
│ incident-management/ testing-and-quality/                      │
│                                                                 │
│ Purpose: Platform operations, reliability, GenAI at scale      │
├─────────────────────────────────────────────────────────────────┤
│ Layer 2: Build Reliable Services                                │
│ backend-engineering/ frontend-engineering/ databases/          │
│ data-engineering/ security/ testing-and-quality/               │
│                                                                 │
│ Purpose: Core engineering skills, security, data              │
├─────────────────────────────────────────────────────────────────┤
│ Layer 1: Understand the Domain                                 │
│ banking-domain/ regulations-and-compliance/                    │
│ engineering-philosophy/ onboarding/ glossary/                  │
│                                                                 │
│ Purpose: Domain knowledge, mindset, foundations                │
└─────────────────────────────────────────────────────────────────┘
```

## Cross-Reference Design

Sections are designed to be interconnected. Key cross-reference patterns:

### Vertical References (Layer-to-Layer)

Every technical guide references:
- **Layer 1:** Why this matters in banking/regulated contexts
- **Layer 4:** How to design, lead, and interview on this topic

Example: `security/prompt-injection.md` →
- `banking-domain/` (why it matters for banks)
- `regulations-and-compliance/` (compliance implications)
- `genai-platforms/ai-safety.md` (GenAI safety context)
- `system-design/secure-rag-platform.md` (design implications)
- `interview-prep/security-interview-questions.md` (interview prep)

### Horizontal References (Same Layer)

Related topics reference each other:

Example: `rag-and-search/retrieval.md` →
- `databases/pgvector.md` (vector DB implementation)
- `genai-platforms/rag.md` (RAG framework context)
- `skills/rag-evaluation.md` (quality evaluation)
- `backend-engineering/api-design/` (API design patterns)

## Content File Design

Every content file follows a consistent structure:

```markdown
# Topic Title

## Core Principles
## Mental Models
## Step-by-Step Approach
## Common Pitfalls
## Banking-Specific Concerns
## GenAI-Specific Concerns
## Metrics to Monitor
## Interview Questions
## Hands-On Exercise
## Related Files
```

This ensures predictability and reusability across the handbook.

## Extension Points

The handbook is designed to grow. New content can be added by:

1. **Adding files to existing sections** — Follow the file design pattern above
2. **Adding new subdirectories** — For specialized topics within a section
3. **Adding new sections** — Requires a feature request and maintainer approval

### Suggested Future Additions

- `healthcare-domain/` — HIPAA, healthcare AI, clinical decision support
- `government-domain/` — FedRAMP, government AI, sovereign AI
- `insurance-domain/` — Actuarial AI, claims processing, risk modeling
- `advanced-ml/` — Traditional ML, model training, feature engineering
- `ethics/` — AI ethics, fairness, bias mitigation deep-dive
- `legal/` — AI law, IP, liability, regulatory landscape

## GitHub Automation Design

The CI pipeline validates content quality:

```
PR Submitted
    │
    ▼
┌──────────────────┐
│ Markdown Lint    │  ← Formatting, style, structure
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Spellcheck       │  ← Spelling, technical terms
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Link Check       │  ← Broken or outdated links
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Structure Check  │  ← Directory structure, file count
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Human Review     │  ← Content, accuracy, quality
└────────┬─────────┘
         │
         ▼
    Merged → main
```

---

For contribution guidelines, see [CONTRIBUTING.md](CONTRIBUTING.md).
For the development roadmap, see [ROADMAP.md](ROADMAP.md).
