# Frontend Engineering — React, Next.js, TypeScript

> Building safe, performant, accessible enterprise UIs for a global bank's GenAI platform.

## Overview

This section covers the frontend engineering practices for building the internal GenAI assistant platform used by hundreds of thousands of bank employees. Our frontend must meet the same rigorous standards as our backend: security-first, fully auditable, highly accessible, and performant under real-world conditions.

### What We Build

- **Internal GenAI Assistant** — Chat interface for employees to query policies, draft communications, analyze data, and automate tasks
- **Compliance Dashboards** — Real-time monitoring of AI guardrails, usage metrics, and policy violations
- **Admin Portals** — Model management, prompt template configuration, user access control
- **Human Review Interfaces** — Workflows for reviewing AI-generated content before external release
- **Audit & Observability UIs** — Searchable logs, conversation history, feedback analysis

## Tech Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Framework | **Next.js 15 (App Router)** | Server rendering, streaming, API routes, mature ecosystem |
| Language | **TypeScript 5.x** | Type safety across large codebases, catch errors at compile time |
| UI Library | **React 19** | Concurrent features, Server Components, Suspense |
| State (Server) | **TanStack React Query v5** | Caching, deduplication, optimistic updates |
| State (Client) | **Zustand** | Lightweight, minimal boilerplate, no providers needed |
| Forms | **React Hook Form + Zod** | Performance, schema validation, type inference |
| Styling | **Tailwind CSS + CSS Variables** | Design token system, dark mode, consistent theming |
| Components | **Internal Design System** | Bank-branded, accessible, WCAG 2.1 AA certified |
| Charts | **Recharts / D3** | Compliance dashboards, usage analytics |
| Testing | **Jest + React Testing Library + Playwright** | Unit, integration, E2E coverage |
| Virtualization | **@tanstack/react-virtual** | Large tables, audit logs, message lists |
| Error Tracking | **Sentry** | Frontend error monitoring, performance tracing |
| Analytics | **Internal telemetry platform** | Usage metrics, feature adoption, funnel analysis |

## Architecture Principles

### 1. Security Is Non-Negotiable

Every component, every API call, every piece of rendered content must be treated as a potential attack surface.

```
User Input → Sanitize → Validate → Render
                ↑
         Content Security Policy
         XSS Prevention
         Input Validation
```

### 2. Performance Is a Feature

Our users include employees on low-bandwidth branch office connections. The UI must feel instant.

**Performance Budgets:**
- First Contentful Paint: < 1.5s
- Time to Interactive: < 3.5s
- Total Bundle Size (initial): < 200KB gzipped
- Largest Contentful Paint: < 2.5s
- Cumulative Layout Shift: < 0.1

### 3. Accessibility Is a Legal Requirement

Banks are subject to accessibility regulations in every jurisdiction we operate in. WCAG 2.1 AA is not a target — it is the minimum.

### 4. Every Feature Is a Two-Way Door

Feature flags gate every new UI capability. We can enable, disable, and roll back without a deployment.

## Team Structure

```
Frontend Platform Team (6-8 engineers)
├── Staff Frontend Engineer (Tech Lead)
│   ├── Architecture decisions, cross-team alignment
│   ├── Performance budgets, design system direction
│   └── Mentoring senior engineers
├── Senior Frontend Engineers (3-4)
│   ├── Feature development (chat, dashboards, admin)
│   ├── Code reviews, testing, documentation
│   └── Accessibility audits
├── Frontend Engineer (1-2)
│   ├── Component development
│   ├── Bug fixes, tests
│   └── Design system contributions
└── UX Engineer (1)
    ├── Design system maintenance
    ├── Accessibility testing
    └── Developer experience tooling
```

### Collaboration Model

```
Frontend Team ──────┬────── Backend Team (APIs, GenAI services)
                    │
                    ├────── Platform Team (CI/CD, hosting, CDN)
                    │
                    ├────── Security Team (reviews, threat modeling)
                    │
                    ├────── Design Team (Figma, tokens, prototypes)
                    │
                    └────── Compliance Team (accessibility, audit sign-offs)
```

## Project Structure

```
frontend/
├── src/
│   ├── app/                          # Next.js App Router
│   │   ├── (auth)/                   # Auth route group
│   │   │   ├── login/
│   │   │   ├── callback/
│   │   │   └── layout.tsx
│   │   ├── (dashboard)/              # Authenticated route group
│   │   │   ├── chat/                 # GenAI chat interface
│   │   │   ├── compliance/           # Compliance dashboards
│   │   │   ├── admin/                # Admin portal
│   │   │   ├── review/               # Human review queues
│   │   │   ├── audit/                # Audit log search
│   │   │   └── layout.tsx            # Dashboard layout (nav, sidebar)
│   │   ├── api/                      # API routes (BFF pattern)
│   │   │   ├── chat/
│   │   │   ├── feedback/
│   │   │   └── health/
│   │   ├── layout.tsx                # Root layout (providers, fonts)
│   │   └── page.tsx                  # Landing / redirect
│   ├── components/                   # Shared components
│   │   ├── ui/                       # Design system primitives
│   │   ├── chat/                     # Chat-specific components
│   │   ├── compliance/               # Compliance components
│   │   ├── forms/                    # Form building blocks
│   │   ├── layout/                   # Navigation, sidebar, header
│   │   └── shared/                   # Cross-cutting utilities
│   ├── hooks/                        # Custom React hooks
│   ├── lib/                          # Utility libraries
│   │   ├── api/                      # API client wrappers
│   │   ├── auth/                     # Auth utilities
│   │   ├── security/                 # Sanitization, validation
│   │   └── utils/                    # Pure utility functions
│   ├── stores/                       # Zustand stores
│   ├── types/                        # Shared TypeScript types
│   ├── config/                       # Feature flags, constants
│   └── styles/                       # Global styles, tokens
├── public/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── next.config.ts
├── tailwind.config.ts
├── tsconfig.json
└── package.json
```

## Key Decisions (ADRs)

### ADR-001: Next.js App Router Over Pages Router

**Decision:** Use Next.js 15 App Router with React Server Components.

**Rationale:**
- Server Components reduce client-side JavaScript by 30-50%
- Built-in streaming SSR improves perceived performance
- API routes provide a Backend-for-Frontend (BFF) layer
- Mature ecosystem, backed by Vercel

**Tradeoffs:**
- Learning curve for Server vs Client component boundaries
- Some third-party libraries not yet Server Component compatible

### ADR-002: React Query Over Redux for Server State

**Decision:** Use TanStack React Query for all server state management.

**Rationale:**
- Automatic caching, deduplication, and refetching
- Built-in loading, error, and success states
- DevTools for debugging server state
- Eliminates boilerplate of Redux for API data

**Tradeoffs:**
- Team must learn new mental model (queries vs actions)
- Complex mutations require careful optimistic update design

### ADR-003: Internal Design System Over Third-Party

**Decision:** Build and maintain an internal component library.

**Rationale:**
- Full control over accessibility compliance
- Bank-specific branding and theming requirements
- No external dependency on component API changes
- Security audit of every component

**Tradeoffs:**
- Upfront investment of 2-3 engineer-months
- Ongoing maintenance burden

## Common Mistakes

1. **Putting business logic in components**: Components should be UI-only. Use hooks and services for data transformation.
2. **Ignoring accessibility until late**: Retrofitting a11y costs 10x more than building it in from day one.
3. **Shipping oversized bundles**: Every KB of JavaScript is parse/compile time. Audit bundle size weekly.
4. **Storing sensitive data in client state**: Tokens, PII, and secrets must never live in Zustand, Context, or localStorage.
5. **No error boundary strategy**: A single component crash should not white-screen the entire application.
6. **Skipping performance budgets**: Without explicit budgets, the app gradually slows until users complain.
7. **Feature flags as afterthought**: Every feature should launch behind a flag from day one.

## Interview Questions

1. How do you decide between a Server Component and a Client Component in Next.js?
2. Design a chat interface that handles streaming responses with proper loading states.
3. How would you implement role-based UI rendering for a compliance dashboard?
4. What is your strategy for preventing XSS when rendering AI-generated markdown content?
5. How do you measure and enforce performance budgets in a CI/CD pipeline?
6. Explain how you would test accessibility in a React application.
7. Design an optimistic update flow for a feedback submission form.

## Cross-References

- `../backend-engineering/typescript/` — TypeScript backend patterns
- `../genai-platforms/` — GenAI platform architecture
- `../security/` — Security engineering practices
- `./component-architecture.md` — Component design patterns
- `./state-management.md` — State management strategy
- `./secure-frontend-patterns.md` — Frontend security patterns
- `./accessibility.md` — Accessibility requirements
- `./performance-optimization.md` — Performance budgets and techniques
- `./genai-chat-interfaces.md` — Chat UI architecture
- `./streaming-responses.md` — Streaming response handling

## Getting Started

1. Clone the frontend repository
2. Run `npm install` to install dependencies
3. Copy `.env.example` to `.env.local` and configure variables
4. Run `npm run dev` to start the development server
5. Run `npm run test` to execute the test suite
6. Run `npm run lint` to check code quality

```bash
# Development
npm run dev

# Build for production
npm run build

# Run all tests
npm run test

# Run E2E tests
npm run test:e2e

# Analyze bundle size
npm run analyze

# Accessibility audit
npm run a11y:audit
```

---

*"The frontend is what users touch. If it breaks, the entire platform feels broken — no matter how solid the backend is."*
