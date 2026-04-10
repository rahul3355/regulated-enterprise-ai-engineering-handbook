# Changelog

All notable changes to this handbook will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- Initial release of the complete handbook

---

## [1.0.0] — 2025-04-10

### Added

#### Core Structure (29 sections, 544 files)

**Engineering Foundations:**
- `onboarding/` — 9 files: 30/60/90 day plans, platform understanding, regulated environments
- `engineering-philosophy/` — 11 files: Bias for action, craftsmanship, pragmatism vs excellence
- `engineering-culture/` — 16 files: Code reviews, mentorship, pair programming, psychological safety
- `leadership-and-collaboration/` — 15 files: Influence, stakeholder management, executive communication

**Domain Knowledge:**
- `banking-domain/` — 13 files: Banking 101 through treasury and risk
- `regulations-and-compliance/` — 21 files: GDPR, PCI-DSS, SOC2, ISO 27001, BCBS 239, SR 11-7, AI governance
- `glossary/` — 11 files: 250+ terms across all domains

**Technical Engineering:**
- `backend-engineering/` — 36+ files: Python/FastAPI, Go, Java/Spring, TypeScript/NestJS + 10 pattern guides
- `frontend-engineering/` — 19 files: React, Next.js, streaming AI UIs, accessibility, safe content rendering
- `databases/` — 23 files: Postgres deep-dive, pgvector, Redis, replication, HA, multi-tenant patterns
- `data-engineering/` — 26 files: SQL, Kafka, Spark, CDC, data quality, embedding pipelines
- `security/` — 24 files: OWASP, OAuth2, prompt injection, jailbreaks, supply chain, GenAI threat modeling
- `testing-and-quality/` — 22 files: Unit through E2E, chaos engineering, LLM evaluation, red teaming

**AI & GenAI:**
- `genai-platforms/` — 34+ files: LLM fundamentals through safe rollouts, 6 banking use cases, 13 sub-framework guides
- `rag-and-search/` — 24 files: Complete RAG academy from chunking to production incidents

**Platform & Operations:**
- `kubernetes-openshift/` — 23 files: Full K8s/Ops guide including GPU workloads and secure deployments
- `cicd-devops/` — 21 files: GitHub Actions, Tekton, Harness, Terraform, change management
- `observability/` — 22 files: OpenTelemetry, SLOs, GenAI-specific monitoring, hallucination detection
- `infrastructure/` — 13 files: Cloud, networking, platform engineering, golden paths
- `incident-management/` — 18 files: Incident command, communication, GenAI playbooks, banking obligations

**Architecture & Design:**
- `architecture/` — 23 files: ADRs, patterns, 3 full architecture examples
- `system-design/` — 15 files: 14 complete system designs with interview Q&A

**Practice & Preparation:**
- `interview-prep/` — 18 files: 300+ questions, STAR stories, exercises, mock interviews
- `exercises/` — 22 files: Hands-on challenges with solutions and extensions
- `templates/` — 17 files: Design docs, RFCs, ADRs, postmortems, runbooks
- `real-world-case-studies/` — 10 files: Detailed production incident stories
- `agents/` — 13 files: AI-assisted engineering role instructions
- `skills/` — 18 files: Reusable engineering checklists and mental models

#### GitHub Infrastructure
- CI/CD workflows: markdown lint, spellcheck, link checking, combined CI
- Issue templates: Bug report, feature request, documentation, security, question
- PR template with security, compliance, and production readiness checks
- CODEOWNERS for review routing
- Dependabot configuration
- CONTRIBUTING.md, CODE_OF_CONDUCT.md, SECURITY.md, SUPPORT.md

---

## Versioning Guidance

This handbook uses **Semantic Versioning** for released versions:

- **MAJOR** — Significant content reorganization, new major sections, or structural changes
- **MINOR** — New files within existing sections, substantial content additions
- **PATCH** — Corrections, clarifications, link fixes, typo fixes

The `main` branch always contains the latest content. Released versions are tagged.

For detailed contribution and release processes, see [CONTRIBUTING.md](CONTRIBUTING.md).

[Unreleased]: https://github.com/USERNAME/regulated-enterprise-ai-engineering-handbook/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/USERNAME/regulated-enterprise-ai-engineering-handbook/releases/tag/v1.0.0
