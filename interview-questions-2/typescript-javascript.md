# TypeScript & JavaScript Interview Questions — 20 Questions with Full Answers

## Overview
| Detail | Value |
|--------|-------|
| Topic | TypeScript/JavaScript (Backend + Frontend) |
| Questions | 20 (5 Must-Know, 10 Medium, 5 Advanced) |
| Source Files | backend-engineering/typescript/ (5 files), frontend-engineering/ (19 files) |
| Citi Relevance | Citi's frontend is TypeScript + React + Next.js; backend uses TypeScript with NestJS/Express. Streaming GenAI UIs require deep React/Next.js expertise. |

---

## Difficulty Legend
- 🔵 **Must-Know** — Foundational. Every candidate should know these.
- 🟡 **Medium** — Core competency. What you'll actually be tested on.
- 🔴 **Advanced** — Differentiator. Shows principal-engineer depth.

---

### Q1: What is the difference between server state and client state in a React application, and why does this distinction matter for a banking GenAI platform?

**Strong Answer:**

Server state and client state are fundamentally different concerns that require different tools. Server state is data that lives on a server, is controlled by someone else (the backend), can become stale, can be updated by other users, and requires network requests to change. In a banking GenAI platform, this includes conversation history, policy documents, compliance reports, user permissions, feature flags, and audit logs.

Client state, by contrast, is data that lives only in the UI, is controlled by the current user, changes instantly, does not need to be persisted, and has no risk of becoming stale. Examples include sidebar collapsed/expanded state, active tab selection, form field values before submission, streaming text buffers, modal open/close status, and toast notifications.

The distinction matters enormously in a regulated banking environment because mixing these two leads to unnecessary complexity, stale data, cache invalidation bugs, and performance issues. For server state, we use TanStack React Query v5 which handles caching, deduplication, background refetching, pagination, and optimistic updates out of the box. For client state, we use Zustand — a lightweight store that requires no providers and supports selective subscriptions to prevent unnecessary re-renders.

In a banking context, storing sensitive data like auth tokens in client state (Zustand, Context, or localStorage) is a critical security vulnerability because it exposes them to XSS attacks. Tokens must be in httpOnly cookies managed by the auth library. The frontend reads auth status from a session endpoint, never the token itself.

**Key Points to Hit:**
- [ ] Server state: controlled by backend, can be stale, needs caching (React Query)
- [ ] Client state: local UI-only, changes instantly, no staleness (Zustand)
- [ ] Examples from banking: policies, audit logs vs. sidebar state, form inputs
- [ ] Security: never store tokens in client-side JavaScript memory
- [ ] Wrong tool choice leads to manual cache invalidation, race conditions, stale data

**Follow-Up Questions:**
1. When would you choose Context over Zustand for client state?
2. How does React Query handle request deduplication when two components use the same query key?
3. Why is `useEffect` for data fetching considered an anti-pattern in modern React?

**Source:** `banking-genai-engineering-academy/frontend-engineering/state-management.md`

---

### Q2: How do you decide between a Server Component and a Client Component in Next.js 15 App Router?

**Strong Answer:**

The rule is simple: every component starts as a Server Component by default. Only add `"use client"` when interactivity is required — that is, when you need hooks, event handlers, useState, useEffect, or browser-only APIs. Server Components render on the server and send zero JavaScript to the client, reducing bundle size by 30-50%.

Server Components are ideal for fetching data, rendering static content, and composing layouts. For example, a PolicySummary component that fetches a policy document and renders it as HTML should be a Server Component. It fetches data server-side, renders HTML, and ships it to the client with no client-side JavaScript overhead.

Client Components are needed for interactive elements: chat input with auto-resizing textareas, streaming message lists that update as tokens arrive, toggles, dropdown menus, and any component that responds to user actions. A ChatInput component needs `"use client"` because it uses useState for the input value, useRef for the textarea, and callbacks for send/stop events.

The composition pattern that works best is to have a Server Component as the page-level shell that fetches data and passes it to Client Components for interactive rendering. This keeps the bulk of JavaScript on the server while giving users rich interactivity where needed.

In a banking context, Server Components also provide a security benefit: sensitive data fetching happens server-side with backend tokens that are never exposed to the browser. The Server Component acts as a Backend-for-Frontend (BFF) layer, authorizing requests with server-side credentials and returning only the sanitized, user-authorized data.

**Key Points to Hit:**
- [ ] Server Components by default — add "use client" only for interactivity
- [ ] Server Components: data fetching, static rendering, zero JS to client
- [ ] Client Components: hooks, event handlers, state, browser APIs
- [ ] Composition: Server Component shell passes data to Client Components
- [ ] Banking security: Server Components keep backend tokens off the client

**Follow-Up Questions:**
1. Can a Server Component import a Client Component? Can a Client Component import a Server Component?
2. How do you pass server-fetched data to a client component that needs it for rendering?
3. What happens if you put "use client" in a component that has no interactivity?

**Source:** `banking-genai-engineering-academy/frontend-engineering/component-architecture.md`

---

### Q3: How do you share types between frontend and backend in a TypeScript monorepo, and why is this important for a banking platform?

**Strong Answer:**

In a TypeScript monorepo, you create a shared types package that both frontend and backend import. This single source of truth for API contracts eliminates the integration bugs that plague teams maintaining separate type definitions. When the backend changes a response shape, the frontend gets a compile-time error instead of a runtime crash.

The shared package exports request types, response types, error types, and domain models. For example, a PaymentRequest interface with fromAccount, toAccount, amount (as a string for Decimal precision), currency, and optional reference. Both the NestJS backend controller and the React frontend import from the same `@bank/shared-types` package.

This is particularly critical in banking because API contract mismatches can lead to serious issues: a payment amount typed as number on one side and string on the other creates rounding errors, compliance reports show incorrect values, and audit trails become unreliable. TypeScript's compile-time type checking catches these before they reach production.

In practice, the shared types package lives at the workspace root. The backend's DTOs and the frontend's API clients both reference these types. When using Zod for runtime validation, you can infer TypeScript types from the schema with `z.infer`, ensuring runtime validation and compile-time types stay in sync. The key is that the schema itself lives in the shared package, so both sides validate against the same rules.

A common mistake is not using `strict: true` in tsconfig, which misses null checks, implicit any, and other bugs that would otherwise be caught at compile time. In a regulated banking environment, `strict: true` is non-negotiable.

**Key Points to Hit:**
- [ ] Shared types package in monorepo — single source of truth for API contracts
- [ ] Both frontend and backend import from same types
- [ ] Prevents integration bugs: compile-time errors instead of runtime crashes
- [ ] Banking impact: mismatched payment types cause financial and compliance errors
- [ ] Use `strict: true` in tsconfig; use Zod + `z.infer` for runtime/compile-time sync

**Follow-Up Questions:**
1. How do you handle API versioning when types evolve over time?
2. What happens when the backend adds a new optional field — does the frontend need to change?
3. How do you prevent the shared types package from becoming a dumping ground?

**Source:** `banking-genai-engineering-academy/backend-engineering/typescript/README.md`, `banking-genai-engineering-academy/backend-engineering/typescript/type-safety.md`

---

### Q4: How does React Query handle caching and what configuration is appropriate for a banking application?

**Strong Answer:**

React Query provides automatic caching, deduplication, background refetching, and garbage collection for server state. When a component calls `useQuery`, React Query checks if the data is already in the cache. If the data is fresh (within `staleTime`), it returns the cached value immediately. If stale, it returns the cached value and refetches in the background. If multiple components request the same query key simultaneously, React Query deduplicates the requests into a single network call.

For a banking GenAI platform, the QueryClient configuration must be conservative and security-conscious. The `staleTime` defaults to 60 seconds — a balance between data freshness and server load. The `gcTime` (garbage collection time) is set to 10 minutes, keeping data available for navigation without consuming excessive memory. Retry logic is configured to never retry authentication failures (401) or permission errors (403), and to retry server errors (500+) a maximum of 2 times with exponential backoff capped at 5 seconds.

Critically, `refetchOnWindowFocus` is set to false. In a banking compliance context, you don't want the application hammering servers every time a user alt-tabs back to the browser tab. `refetchOnReconnect` is true to refresh data when the network comes back. Most importantly, `mutations` have `retry: false` — you never auto-retry mutations in banking because non-idempotent operations (like creating a payment) could execute twice, causing duplicate transactions.

The query key factory pattern ensures consistency across the application. Instead of magic strings like `['audit']` scattered across components, you use `auditKeys.list(filters)` which generates a structured key. This enables precise cache invalidation: `queryClient.invalidateQueries({ queryKey: auditKeys.lists() })` invalidates all audit list queries without nuking the entire cache.

**Key Points to Hit:**
- [ ] Automatic caching, deduplication, background refetching, garbage collection
- [ ] Banking config: staleTime 60s, gcTime 10min, no retry on mutations
- [ ] refetchOnWindowFocus: false to avoid server hammering
- [ ] Never retry mutations in banking — duplicate transaction risk
- [ ] Query key factory pattern for consistent keys and precise invalidation

**Follow-Up Questions:**
1. How do you implement optimistic updates with rollback on failure?
2. What happens when two components mount simultaneously with the same query key?
3. How do you handle pagination for a 10,000-entry audit log?

**Source:** `banking-genai-engineering-academy/frontend-engineering/react-query.md`, `banking-genai-engineering-academy/frontend-engineering/state-management.md`

---

### Q5: How do you structure a NestJS application for a banking service with 10+ modules?

**Strong Answer:**

NestJS brings Java/Spring-like structure to Node.js with dependency injection, modular architecture, decorators, and built-in testing utilities. For a banking GenAI platform with 10+ modules, the service layout follows a strict separation of concerns.

At the root, `main.ts` bootstraps the application with global pipes, filters, and interceptors. A global ValidationPipe with `whitelist: true` strips unknown properties, `forbidNonWhitelisted: true` rejects them entirely, and `transform: true` converts payloads to DTO instances. In production, `disableErrorMessages: true` prevents leaking internal details. A GlobalExceptionFilter standardizes all error responses with a consistent format including error code, message, correlation ID, and timestamp.

Each business domain is its own module: `modules/payments/`, `modules/accounts/`, `modules/ai/`. Each module contains a controller (HTTP layer), service (business logic), repository (data access), DTOs (validation schemas), and entities (database models). The controller handles only HTTP concerns — route mapping, request parsing, response formatting. All business logic lives in the service. This separation is critical because controllers are easy to test with mocked services, and services can be tested without HTTP concerns.

Modules declare their providers and export only what other modules need. For example, PaymentsModule exports PaymentsService so the AI module can check payment status. Circular dependencies between modules are resolved using `forwardRef()` or by introducing a shared module. The root AppModule imports all feature modules and configures global providers like database connections.

In a banking context, every module includes health checks so Kubernetes can determine pod readiness. The ConfigService manages environment-specific configuration (database URLs, API keys, feature flags) without hardcoding secrets. All external service calls use connection-pooled HTTP clients, not per-request instances.

**Key Points to Hit:**
- [ ] Modular structure: each domain has controller, service, repository, DTOs, entities
- [ ] Global pipes: ValidationPipe with whitelist, GlobalExceptionFilter
- [ ] Controllers handle HTTP only; business logic in services
- [ ] Module exports define the public API; avoid circular dependencies
- [ ] Banking: health checks for K8s readiness, ConfigService for secrets

**Follow-Up Questions:**
1. What is the difference between a Guard, Interceptor, and Pipe in NestJS?
2. How do you handle circular dependencies between modules?
3. How do you implement request-scoped services in NestJS?

**Source:** `banking-genai-engineering-academy/backend-engineering/typescript/express-nestjs.md`

---

### Q6: How do you consume Server-Sent Events (SSE) in a React application for streaming GenAI responses?

**Strong Answer:**

SSE is the standard protocol for streaming AI responses from backend to frontend. The model generates tokens one at a time, and the server pushes each token as an SSE event with the format `data: {"delta": {"content": "token"}}`. The frontend consumes this stream using the Fetch API with a ReadableStream reader.

The pattern starts with a custom hook like `useChatStream` that manages the entire streaming lifecycle. When `sendMessage` is called, it immediately adds the user message to state and creates a placeholder for the assistant response. It then calls `fetch('/api/chat/stream')` with `Accept: 'text/event-stream'` header and an AbortController signal for cancellation.

The response body is a ReadableStream accessed via `response.body.getReader()`. A TextDecoder converts binary chunks to text. The hook loops with `reader.read()`, splitting each chunk by newlines and parsing lines that start with `data: `. Each parsed token is accumulated into a string and the state is updated to re-render the UI with the growing response. When `[DONE]` is received, the message is marked complete and telemetry is sent.

Security is paramount: every token update triggers a re-render, and the content must be sanitized before rendering. Using `useMemo` around `sanitizeAndRenderMarkdown` ensures sanitization runs on every content change but not on unrelated re-renders. Without this, malicious content in the stream could execute XSS attacks.

The AbortController enables the "stop generating" feature. When the user clicks stop, `abortController.abort()` is called, which throws an AbortError in the fetch. The catch handler detects this and finalizes the message with partial content rather than treating it as an error. The hook also tracks time-to-first-token for performance telemetry — a critical metric for GenAI user experience.

**Key Points to Hit:**
- [ ] Fetch API with ReadableStream reader and TextDecoder for SSE consumption
- [ ] Accumulate tokens in state, update UI incrementally
- [ ] AbortController for cancellation ("stop generating" button)
- [ ] Sanitize every render — streamed content is a XSS attack surface
- [ ] Track time-to-first-token and total duration for telemetry

**Follow-Up Questions:**
1. How do you handle malformed JSON in the SSE stream?
2. What happens if the user navigates away while streaming?
3. How do you measure and report streaming performance metrics?

**Source:** `banking-genai-engineering-academy/frontend-engineering/streaming-responses.md`

---

### Q7: How do you implement a query key factory pattern and why does it matter in a large React application?

**Strong Answer:**

A query key factory is a structured object that generates consistent query keys for every domain in your application. Instead of scattering magic strings like `['audit']`, `['audit-logs']`, and `['logs']` across dozens of components, you define a single factory that produces predictable, hierarchical keys.

For example, `auditKeys.all` returns `['audit']`, `auditKeys.lists()` returns `['audit', 'list']`, `auditKeys.list(filters)` returns `['audit', 'list', filters]`, and `auditKeys.detail(id)` returns `['audit', 'detail', id]`. Every component in the application uses these factory methods, guaranteeing that the same logical query always uses the same cache key.

The critical benefit is precise cache invalidation. When a new audit log entry is created, you call `queryClient.invalidateQueries({ queryKey: auditKeys.lists() })` which invalidates all audit list queries regardless of their filter parameters. If you used inconsistent strings, some components would show stale data because their cache key didn't match the invalidation target.

In a banking platform with hundreds of components, this pattern prevents subtle cache bugs that are extremely hard to debug. A compliance officer might see outdated audit logs because one component used `['auditLogs']` while the invalidation targeted `['audit', 'list']`. With a factory, this class of bug is impossible.

The factory also enables advanced patterns like bulk invalidation. Calling `queryClient.invalidateQueries({ queryKey: conversationKeys.all })` invalidates every conversation-related query — lists, details, messages — in a single call. But the preferred approach is surgical: invalidate only what changed to minimize refetching.

**Key Points to Hit:**
- [ ] Factory generates consistent, hierarchical query keys
- [ ] Prevents cache bugs from inconsistent key strings across components
- [ ] Enables precise cache invalidation at different granularities
- [ ] Critical in banking: stale compliance data is a regulatory risk
- [ ] Supports bulk invalidation but surgical invalidation is preferred

**Follow-Up Questions:**
1. How do you include complex filter objects in query keys without causing serialization issues?
2. What happens if two developers add conflicting keys to the factory?
3. How do you test that invalidation targets are correct?

**Source:** `banking-genai-engineering-academy/frontend-engineering/react-query.md`, `banking-genai-engineering-academy/frontend-engineering/state-management.md`

---

### Q8: Design a type-safe API client that enforces request and response types at compile time.

**Strong Answer:**

A type-safe API client uses TypeScript generics and branded types to ensure that every API call has the correct request body, path parameters, and response type — all enforced at compile time. The foundation is a generic route handler type: `type RouteHandler<ReqBody, ReqParams, ResBody>` that takes the request body type, request parameters type, and response body type as generic parameters.

For example, a `createPayment` handler is typed as `RouteHandler<PaymentRequest, {}, PaymentResponse>`. TypeScript will error if the handler tries to return a response that doesn't match `PaymentResponse`, or if it accesses a request body field that isn't in `PaymentRequest`. This catches mismatches like sending a string amount when the API expects a Decimal string representation.

Branded types add another layer of safety. By creating `type AccountId = Branded<string, 'AccountId'>` and `type PaymentId = Branded<string, 'PaymentId'>`, you prevent mixing up different string identifiers at compile time. A function expecting `AccountId` will reject a `PaymentId` even though both are strings underneath. In a banking context, this prevents catastrophic bugs like querying the payments table with an account ID.

On the client side, you can build a generic API client function: `async function apiCall<Res>(endpoint: APIPath, options: RequestInit): Promise<Res>` where `APIPath` is a template literal type like `` `/api/${'v1' | 'v2'}/${'payments' | 'accounts'}` ``. This ensures that only valid API paths can be constructed — `/api/v3/payments` would be a compile error.

For runtime safety, combine compile-time types with Zod validation. Define a Zod schema for each request type, infer the TypeScript type from it with `z.infer`, and validate incoming data at the API boundary. This way, TypeScript types and runtime validation are guaranteed to stay in sync because they derive from the same source.

**Key Points to Hit:**
- [ ] Generic RouteHandler type enforces request/response shapes at compile time
- [ ] Branded types prevent mixing different string IDs (AccountId vs PaymentId)
- [ ] Template literal types enforce valid API paths
- [ ] Zod schemas + z.infer keep runtime validation and types in sync
- [ ] Banking: type-safe APIs prevent financial data corruption

**Follow-Up Questions:**
1. How do you handle API responses that have different shapes based on status codes?
2. What is the performance cost of branded types at runtime?
3. How do you version API types when the backend evolves?

**Source:** `banking-genai-engineering-academy/backend-engineering/typescript/type-safety.md`

---

### Q9: How do you implement optimistic updates with rollback, and when should you NOT use them in a banking application?

**Strong Answer:**

Optimistic updates improve perceived performance by updating the UI immediately, before the server confirms the change. React Query supports this through the `onMutate`, `onError`, and `onSettled` callbacks. The pattern is: cancel any in-flight requests, snapshot the current cache state, apply the optimistic update, and on error, roll back to the snapshot.

For a low-risk operation like submitting feedback on an AI response, optimistic updates are ideal. When the user clicks thumbs-up, the UI immediately increments the counter. If the server call fails, the counter rolls back. The user experience is snappy, and the rollback is invisible.

The implementation uses `onMutate` to capture the previous state: `const previousSummary = queryClient.getQueryData(feedbackKeys.summary())`, then updates the cache with the optimistic value. If the mutation fails, `onError` restores the previous state. Finally, `onSettled` invalidates the query to ensure the server's authoritative state is reflected.

However, for compliance-critical operations like updating a policy document, you must NOT use optimistic updates. Policy changes in a banking environment have regulatory implications — an optimistic update that shows a policy as "approved" before the server confirms it could mislead compliance officers into thinking a change is live when it is still pending. In these cases, you wait for server confirmation before updating the UI, showing a loading state in the meantime.

The general rule is: optimistic updates are safe for user-preference changes, feedback, UI state, and read-heavy operations. They are dangerous for financial transactions, compliance records, access control changes, and any operation that affects audit trails. In a banking GenAI platform with 500K+ employees, the cost of a wrong optimistic update far exceeds the UX benefit.

**Key Points to Hit:**
- [ ] onMutate: cancel queries, snapshot cache, apply optimistic update
- [ ] onError: rollback to snapshot on failure
- [ ] onSettled: invalidate query to sync with server
- [ ] Safe for: feedback, preferences, UI state
- [ ] Unsafe for: policy changes, financial transactions, compliance records, access control

**Follow-Up Questions:**
1. How do you handle concurrent optimistic updates to the same data?
2. What if the server returns a different value than what you optimistically set?
3. How do you show a loading state for non-optimistic mutations?

**Source:** `banking-genai-engineering-academy/frontend-engineering/react-query.md`

---

### Q10: How do you detect and fix event loop blocking in a Node.js backend service?

**Strong Answer:**

The Node.js event loop is single-threaded, meaning any synchronous CPU-bound work blocks all other requests. In a banking API handling thousands of concurrent connections, even a 50ms block can cascade into timeouts across the entire service.

Detection starts with `perf_hooks.monitorEventLoopDelay()`, which tracks event loop delay percentiles. You enable the monitor and check the P99 delay every second. If P99 exceeds 100ms, you log a warning. For more detailed diagnostics, `perf_hooks.eventLoopUtilization()` measures how much of the event loop's time is spent actively processing versus idle. A readiness check can use this to report "degraded" status when utilization exceeds 90%.

Common causes of event loop blocking include: `JSON.parse/stringify` on large objects (common when processing large API responses or audit logs), synchronous file I/O, regex with catastrophic backtracking, large array sorting, and CPU-intensive computation like ML inference for document classification.

The fix depends on the cause. For CPU-intensive work like risk score computation or ML inference, offload to Worker Threads. You create a pool of workers, chunk the data, and distribute computation across threads. The main thread remains responsive to HTTP requests while workers handle the heavy lifting. For large JSON operations, use `fast-json-stringify` with pre-compiled schemas, which is 2-5x faster than native `JSON.stringify`.

For production monitoring, you set up a health endpoint that checks event loop utilization and reports to your load balancer. If the event loop is overloaded, the readiness check returns "degraded" and the load balancer stops routing new traffic to that pod. Node.js memory is configured with `NODE_OPTIONS="--max-old-space-size=4096"` for a 4GB heap, preventing OOM crashes under load.

**Key Points to Hit:**
- [ ] Monitor event loop delay with perf_hooks at P99
- [ ] eventLoopUtilization for readiness checks
- [ ] Causes: JSON on large objects, sync I/O, regex backtracking, heavy computation
- [ ] Fix CPU work with Worker Threads; fix JSON with fast-json-stringify
- [ ] Configure max-old-space-size for production containers

**Follow-Up Questions:**
1. How do you size a Worker Thread pool for variable workloads?
2. What is the difference between cluster mode and worker threads?
3. How do you profile a production Node.js service without restarting it?

**Source:** `banking-genai-engineering-academy/backend-engineering/typescript/performance.md`, `banking-genai-engineering-academy/backend-engineering/typescript/README.md`

---

### Q11: How do you structure a NestJS validation pipeline for a complex payment request?

**Strong Answer:**

NestJS validation uses class-validator decorators on DTOs combined with a global ValidationPipe. For a complex payment request, the DTO defines every field with its constraints, and NestJS automatically validates and transforms incoming requests before they reach the controller.

The `CreatePaymentDto` uses decorators like `@IsString()`, `@IsNotEmpty()`, `@Length(10, 34)` for account numbers, `@IsNumber()`, `@Min(0.01)`, `@Max(1000000)` for amounts, and `@Matches(/^[A-Z]{3}$/)` for currency codes. Optional fields like reference use `@IsOptional()`. Each decorator also has an `@ApiProperty` decorator for Swagger documentation, so the API documentation stays in sync with the validation rules.

The global ValidationPipe in `main.ts` is configured with `whitelist: true` to strip unknown properties, `forbidNonWhitelisted: true` to reject requests with extra fields (important for security — prevents field injection attacks), `transform: true` to convert plain objects to DTO class instances, and `disableErrorMessages: true` in production to avoid leaking internal details.

The controller receives the validated DTO as a typed parameter: `async create(@Body() createPaymentDto: CreatePaymentDto)`. At this point, the data is guaranteed to match the schema — the controller and service can trust the types without additional validation.

For cross-field validation (like ensuring `fromAccount` and `toAccount` are different), you use a custom class-validator at the service level or a custom validator decorator at the DTO level. The service layer adds business validation: checking account existence, verifying sufficient funds, and enforcing daily transaction limits. These checks throw domain-specific exceptions like `UnprocessableEntityException` with structured error codes (`INSUFFICIENT_FUNDS`, `SELF_PAYMENT`) that the GlobalExceptionFilter translates into user-friendly API responses.

In a banking context, the validation pipeline also enforces regulatory limits: maximum transaction amounts, sanctioned country checks, and compliance flags. These rules live in the service layer because they depend on external data (sanctions lists, customer risk profiles) that class-validator decorators cannot access.

**Key Points to Hit:**
- [ ] class-validator decorators on DTO for field-level validation
- [ ] Global ValidationPipe: whitelist, forbidNonWhitelisted, transform
- [ ] Controller receives validated, typed DTO — no manual validation needed
- [ ] Business validation in service: account checks, funds, regulatory limits
- [ ] Structured error codes mapped to user-friendly messages by exception filter

**Follow-Up Questions:**
1. How do you validate nested objects in a DTO?
2. How do you implement custom async validation (e.g., check if account exists)?
3. How do you handle different validation rules for different user roles?

**Source:** `banking-genai-engineering-academy/backend-engineering/typescript/express-nestjs.md`

---

### Q12: How do you implement role-based UI rendering for a compliance dashboard in a banking application?

**Strong Answer:**

Role-based UI rendering in a banking GenAI platform controls what users see based on their security classification and permissions. The implementation uses a Context Provider for stable role data combined with conditional rendering at the component level.

The `SecurityClassificationProvider` wraps the application and provides the user's classification level (public, internal, confidential, restricted) along with a `canAccess(requiredLevel)` helper. The classification levels are numeric (public=0, internal=1, confidential=2, restricted=3), and access is granted when the user's level meets or exceeds the required level.

At the component level, you use the `useSecurityClassification` hook to conditionally render UI elements. A compliance dashboard might show different sections to different roles: all users see general policy summaries, but only users with "confidential" clearance see the sanctions list management panel, and only "restricted" users see the incident investigation tools.

Feature flags provide an additional layer of control. The `FeatureFlagProvider` receives flags from a server endpoint and the `useFeatureFlag` hook checks individual flags like `newChatInterface`, `citationDisplay`, and `humanReviewFlow`. This enables gradual rollout: you can enable the new compliance dashboard for 10% of users in the compliance team before a full rollout.

The critical security principle is that role checks happen server-side for data access and client-side for UI presentation. The server must never send restricted data to unauthorized clients, even if the client hides it. The client-side role check is purely for user experience — it prevents confusion by not showing UI for features the user cannot access.

In Next.js, the BFF pattern (Backend-for-Frontend) in API routes checks the user's session and forwards only the data they are authorized to see. The Server Component renders the authorized data, and the Client Component provides interactive elements. This defense-in-depth approach ensures that a compromised client cannot access restricted data through API manipulation.

**Key Points to Hit:**
- [ ] SecurityClassificationProvider with numeric levels and canAccess helper
- [ ] Conditional rendering at component level based on user role
- [ ] Feature flags for gradual rollout of new UI features
- [ ] Server-side authorization for data, client-side for UI presentation
- [ ] Defense-in-depth: BFF API routes check permissions before returning data

**Follow-Up Questions:**
1. How do you handle role changes mid-session?
2. What happens when a user has multiple roles (e.g., both compliance officer and admin)?
3. How do you test role-based rendering comprehensively?

**Source:** `banking-genai-engineering-academy/frontend-engineering/component-architecture.md`, `banking-genai-engineering-academy/frontend-engineering/state-management.md`

---

### Q13: What is your strategy for preventing XSS when rendering AI-generated markdown content?

**Strong Answer:**

XSS prevention is the single most important security concern when rendering AI-generated content in a GenAI chat interface. The LLM produces arbitrary markdown and HTML, and any unsanitized content in a `dangerouslySetInnerHTML` call is an immediate XSS vulnerability.

The strategy has multiple layers. First, at the markdown parsing level, you use a sanitizer like DOMPurify or a dedicated markdown sanitizer that strips dangerous tags (`<script>`, `<iframe>`, `<object>`, `<embed>`) and event handlers (`onerror`, `onclick`, `onload`). It also removes `javascript:` URIs in links, which are a common XSS vector. The sanitizer allows safe formatting tags (`<strong>`, `<em>`, `<code>`, `<pre>`, `<blockquote>`, `<ul>`, `<ol>`, `<li>`, `<h1>`-`<h6>`) so legitimate markdown formatting renders correctly.

Second, you sanitize on every render, not just once. In streaming responses, the content changes with each token, so the `StreamingMessageContent` component uses `useMemo(() => sanitizeAndRenderMarkdown(content), [content])` to re-sanitize whenever the content updates. This prevents an attack where the first tokens are safe but later tokens inject malicious content.

Third, Content Security Policy (CSP) headers provide a defense-in-depth layer. Even if sanitization fails, CSP prevents inline script execution and restricts the sources from which scripts can load. The banking platform sets strict CSP headers that disallow `unsafe-inline` scripts and only permit scripts from the application's own domain.

Fourth, you limit the maximum message length to prevent abuse through oversized payloads. If the accumulated content exceeds `MAX_MESSAGE_LENGTH`, you truncate the message and stop processing. This prevents both denial-of-service through memory exhaustion and attacks that hide malicious content in extremely long messages.

The test suite validates all of this: unit tests confirm that script tags, event handlers, javascript URIs, and iframe tags are stripped, while legitimate formatting is preserved. Playwright E2E tests verify that the chat UI renders safely with realistic AI-generated content.

**Key Points to Hit:**
- [ ] DOMPurify or equivalent sanitizer strips dangerous tags and event handlers
- [ ] Sanitize on every render — streaming content changes with each token
- [ ] Content Security Policy headers as defense-in-depth
- [ ] Maximum message length to prevent DoS and hidden attacks
- [ ] Automated tests: unit tests for sanitizer, E2E for safe rendering

**Follow-Up Questions:**
1. How do you handle code blocks with language-specific syntax highlighting safely?
2. What if the AI generates markdown that includes internal URLs that should be clickable?
3. How do you audit your sanitization configuration for completeness?

**Source:** `banking-genai-engineering-academy/frontend-engineering/streaming-responses.md`, `banking-genai-engineering-academy/frontend-engineering/frontend-testing.md`, `banking-genai-engineering-academy/frontend-engineering/README.md`

---

### Q14: How do you implement error boundaries in a React application and what is the difference between route-level and component-level boundaries?

**Strong Answer:**

Error boundaries are React's last line of defense when components crash. In a banking application, a single component failure must never white-screen the entire application or expose stack traces to users. Error boundaries catch JavaScript errors in their child component tree, log them, and display a fallback UI.

Next.js provides built-in route-level error boundaries via `error.tsx` files in route segments. When any component in that route throws, Next.js renders the error.tsx file instead. This is the broadest boundary — it catches everything in the route. For banking, the error.tsx logs to Sentry with feature context, shows a user-friendly message, and provides "Try Again" and "Return to Dashboard" buttons. In development mode, it also shows the error message for debugging.

Feature-level boundaries use a custom `ErrorBoundary` class component that wraps major features like the chat interface, compliance dashboard, and admin portal. This isolates failures: if the chat crashes, the sidebar and header remain functional. Each boundary is named (e.g., `name="chat-interface"`) for precise error tracking. You can provide custom fallback UI per feature — the chat shows a simple "Chat is temporarily unavailable" message, while the compliance dashboard shows a more prominent alert.

Component-level boundaries wrap individual complex components that are likely to fail, like data visualizations or markdown renderers. This is the most granular approach — a failed chart doesn't take down the entire compliance dashboard.

The three-tier strategy places boundaries at all three levels. Route boundaries catch unexpected errors, feature boundaries isolate major sections, and component boundaries protect complex widgets. Every caught error is reported to Sentry with feature tags, component stack traces, and user context so the engineering team can prioritize fixes.

Server Components cannot use error boundaries (they are client-only). Instead, Server Components use try/catch in data fetching and re-throw to trigger the route-level `error.tsx`. The `Promise.allSettled` pattern enables partial rendering — if one data source fails, the others still display.

**Key Points to Hit:**
- [ ] Three-tier strategy: route (error.tsx), feature (ErrorBoundary class), component
- [ ] Route boundaries catch everything in the route; feature boundaries isolate sections
- [ ] Every error reported to Sentry with feature context and component stack
- [ ] User-friendly messages only — no stack traces in production
- [ ] Server Components: use try/catch and Promise.allSettled for partial rendering

**Follow-Up Questions:**
1. How do error boundaries differ from try/catch blocks?
2. Can an error boundary catch an error in an event handler?
3. How do you test that error boundaries actually catch errors?

**Source:** `banking-genai-engineering-academy/frontend-engineering/error-boundaries.md`

---

### Q15: How do you enforce performance budgets in a CI/CD pipeline for a Next.js application?

**Strong Answer:**

Performance budgets are hard limits on metrics like bundle size, LCP, CLS, and TTI that prevent the application from gradually degrading over time. Without explicit budgets, the app slows down incrementally — each PR adds a few KB here, a small component there — until users on branch office connections can barely use it.

The enforcement happens at three levels. First, bundle analysis in CI using `@next/bundle-analyzer` generates a report on every build. The `bundlesize` configuration in package.json sets per-chunk limits: main chunk under 50KB, chat route under 80KB, compliance route under 100KB. If any chunk exceeds its budget, the CI build fails. This catches oversized dependencies before they reach production.

Second, Web Vitals monitoring runs in production via `next/web-vitals`. The `useReportWebVitals` hook sends FCP, LCP, CLS, INP, and TTFB metrics to the telemetry platform on every page load. A budget checker compares each metric against thresholds (FCP < 1.5s, LCP < 2.5s, CLS < 0.1, INP < 200ms) and alerts the team when real-user measurements violate budgets. This catches performance regressions that only appear in production due to network conditions or device capabilities.

Third, Lighthouse CI runs automated performance audits on every PR against a staging deployment. It checks Core Web Vitals, accessibility scores, and best practices. The CI configuration sets minimum score thresholds (performance >= 90, accessibility >= 95) and fails the PR if any threshold is not met.

Next.js automatic code splitting helps meet these budgets by creating separate JavaScript chunks per route. Dynamic imports with `next/dynamic` further reduce initial bundle size by lazy-loading heavy components like charts and markdown renderers. The `ssr: false` option skips server rendering for canvas-based charts that cannot render server-side anyway.

In a banking context, performance is not just a UX concern — it is an accessibility and compliance concern. Slow interfaces disproportionately affect users on low-bandwidth branch office connections, and inaccessible performance patterns can violate WCAG requirements.

**Key Points to Hit:**
- [ ] Bundle analysis in CI with per-chunk size budgets (bundlesize)
- [ ] Web Vitals in production with useReportWebVitals and budget alerts
- [ ] Lighthouse CI on every PR with minimum score thresholds
- [ ] Next.js automatic code splitting + dynamic imports for lazy loading
- [ ] Performance is an accessibility/compliance concern in banking

**Follow-Up Questions:**
1. How do you handle a legitimate feature that requires exceeding a budget?
2. What is the difference between lab metrics (Lighthouse) and field metrics (RUM)?
3. How do you attribute a performance regression to a specific PR?

**Source:** `banking-genai-engineering-academy/frontend-engineering/performance-optimization.md`

---

### Q16: How do discriminated unions improve type safety in TypeScript, and how do they apply to a banking payment processing system?

**Strong Answer:**

Discriminated unions (also called tagged unions or algebraic data types) are a TypeScript pattern where a union of types shares a common literal property — the discriminant — that allows TypeScript to narrow the type in conditional blocks. This eliminates runtime type checking and provides compile-time exhaustiveness checking.

In a banking payment system, different payment methods have different required fields. A wire transfer needs `swiftCode`, `beneficiaryName`, and `beneficiaryAccount`. A SEPA transfer needs `iban`, `beneficiaryName`, and optionally `bic`. A domestic transfer needs `accountNumber`, `routingNumber`, and `beneficiaryName`. A discriminated union models this precisely:

```typescript
type PaymentMethod = WireTransfer | SEPATransfer | DomesticTransfer;
```

Each variant has a `type` property with a literal value (`'wire'`, `'sepa'`, `'domestic'`). When you switch on `method.type`, TypeScript narrows the type in each case branch. In the `case 'wire':` branch, TypeScript knows `method` is a `WireTransfer` and allows access to `method.swiftCode` without any type assertion. In `case 'sepa':`, it knows `method` is a `SEPATransfer`.

The critical safety benefit is exhaustiveness checking. If you add a new payment method like `InstantPayment`, TypeScript will error on the switch statement until you handle the new case. In a banking system where missing a payment type could mean processing it incorrectly (sending an instant payment through the wire transfer pipeline), this compile-time guarantee prevents production incidents.

Combined with branded types for domain identifiers, the type system enforces that a `processPayment` function can only be called with a valid `PaymentMethod`, and each processing path receives exactly the fields it needs. No runtime type guards, no `any` casts, no "this should never happen" comments. The types guarantee correctness.

**Key Points to Hit:**
- [ ] Discriminated union: union types sharing a common literal discriminant property
- [ ] TypeScript narrows type in switch/case branches — no type assertions needed
- [ ] Exhaustiveness checking: adding a new variant forces handling all cases
- [ ] Banking: prevents processing wrong payment type through wrong pipeline
- [ ] Combined with branded types for complete type safety

**Follow-Up Questions:**
1. How do you implement exhaustiveness checking at runtime (not just compile time)?
2. Can discriminated unions work with interfaces, or do they need type aliases?
3. How do you handle API responses that should map to discriminated unions?

**Source:** `banking-genai-engineering-academy/backend-engineering/typescript/type-safety.md`

---

### Q17: How do you virtualize a message list with 10,000+ entries in a React chat application, and what are the trade-offs?

**Strong Answer:**

Virtualization (or windowing) renders only the items visible in the viewport plus a small buffer, instead of creating DOM nodes for all 10,000 items. For a banking compliance audit log or a long-running GenAI conversation history, this is essential — rendering 10,000 message bubbles would consume hundreds of megabytes of DOM and cause severe jank.

The implementation uses `@tanstack/react-virtual`'s `useVirtualizer` hook. You provide the total item count, a reference to the scroll container, an estimated item height, and an overscan value (extra items to render above and below the viewport). The virtualizer calculates which items are visible and returns only those with their absolute positioning.

The key implementation detail is that each virtualized item must have a stable key (the message ID, not the index) and must use `ref={virtualizer.measureElement}` with `data-index` so the virtualizer can measure actual rendered heights and correct its estimates. Items are positioned absolutely with `transform: translateY(virtualRow.start)` for GPU-accelerated scrolling.

Auto-scrolling to the bottom becomes more complex with virtualization. Instead of `scrollIntoView`, you set `parentRef.current.scrollTop = parentRef.current.scrollHeight`. For infinite scroll, you detect when the user is near the bottom (`scrollHeight - scrollTop - clientHeight < 200`) and trigger `fetchNextPage()`.

The trade-offs are significant. Virtualization breaks browser Find (Ctrl+F) because off-screen items are not in the DOM. It complicates accessibility — screen readers expect all content to be present. You must implement `role="log"` with `aria-live="polite"` and ensure the virtualizer announces new items. Variable-height items require measurement passes that can cause layout shifts if estimates are wrong. And virtualization adds complexity: you cannot use simple `.map()` rendering anymore.

In a banking context, the most important trade-off is audit compliance. If a compliance officer needs to search through 10,000 audit log entries, virtualization means browser search won't find off-screen entries. You must implement a server-side search that returns filtered results, rather than relying on client-side text search.

**Key Points to Hit:**
- [ ] useVirtualizer renders only visible items + overscan buffer
- [ ] Stable keys (message ID), measureElement refs, absolute positioning with transform
- [ ] Auto-scroll via scrollTop = scrollHeight; infinite scroll via near-bottom detection
- [ ] Trade-offs: breaks browser Find, complicates a11y, variable-height complexity
- [ ] Banking: audit compliance requires server-side search, not client-side Find

**Follow-Up Questions:**
1. How do you handle variable-height items without causing layout shifts?
2. How do you implement virtualization with Next.js Server Components?
3. What is the memory saving from virtualizing 10,000 items?

**Source:** `banking-genai-engineering-academy/frontend-engineering/performance-optimization.md`, `banking-genai-engineering-academy/frontend-engineering/react-query.md`

---

### Q18: How do you design a custom hook for managing a multi-step form with validation in a banking compliance workflow?

**Strong Answer:**

A multi-step form in a banking compliance workflow — such as reviewing and approving a policy change — requires careful state management, validation at each step, and the ability to navigate forward and backward without losing data. The hook encapsulates step tracking, validation logic, and form state.

The hook uses `useState` for the current step index, a form data object, and a validation errors map. Each step has a corresponding validation function that checks the fields relevant to that step. When the user clicks "Next," the hook runs the current step's validator. If validation passes, it increments the step; if it fails, it populates the errors map and stays on the current step.

For the actual form implementation, you use React Hook Form with Zod for schema validation. Each step gets its own Zod schema, and the `zodResolver` integrates with React Hook Form. The hook exposes `currentStep`, `totalSteps`, `formData`, `errors`, `goToStep`, `nextStep`, `prevStep`, `updateField`, and `submitForm`.

The critical banking-specific concern is that form data must never be persisted to localStorage or any client-side storage, because compliance forms may contain sensitive policy details, account numbers, or PII. All form state lives in React component state, and if the user navigates away, the data is lost (which is the correct security behavior).

The submit function sends the complete form data to the server in a single mutation. For compliance workflows, you do NOT use optimistic updates — the form waits for server confirmation before showing success. The server validates the entire payload again (defense in depth), creates an audit trail entry, and returns the final status.

Accessibility is essential: each step has a heading, validation errors use `role="alert"` and `aria-live="polite"`, and the step indicator uses `aria-current="step"` so screen reader users know their progress. The hook does not manage accessibility directly — that is the component's responsibility — but it provides the data the component needs.

**Key Points to Hit:**
- [ ] Hook manages: currentStep, formData, errors, navigation, validation per step
- [ ] React Hook Form + Zod per step; zodResolver integration
- [ ] Never persist form data to localStorage in banking — sensitive data risk
- [ ] No optimistic updates on submit — wait for server confirmation and audit trail
- [ ] Accessibility: role="alert" for errors, aria-current for step indicator

**Follow-Up Questions:**
1. How do you handle form state persistence across page refreshes (if required)?
2. How do you implement conditional steps (step 3 appears only if step 2 selects option X)?
3. How do you test a multi-step form hook comprehensively?

**Source:** `banking-genai-engineering-academy/frontend-engineering/component-architecture.md`, `banking-genai-engineering-academy/frontend-engineering/frontend-testing.md`

---

### Q19: How do you handle memory leaks in a Node.js backend service, and what tools do you use for profiling in production?

**Strong Answer:**

Memory leaks in Node.js are insidious because the garbage collector does not report what it retains — it only frees what it can prove is unreachable. A slow leak might take days to manifest in production, making it hard to correlate with a specific deployment.

The first line of defense is monitoring. `process.memoryUsage()` returns `rss` (resident set size), `heapTotal`, `heapUsed`, and `external` memory. Logging these every 60 seconds reveals trends: if `heapUsed` grows continuously without plateauing, you have a leak. An alert threshold at 500MB heap usage catches the problem before OOM crashes.

Common Node.js memory leaks in banking services include: global variables accumulating request data, event listeners not removed after use (especially in WebSocket connections for real-time notifications), closures holding references to large objects, unbounded caches that grow without eviction, and large objects stored in request context that persist beyond the request lifecycle.

For profiling, you use three tools in combination. `node --prof` generates a CPU profile that shows which functions consume the most time. `node --prof-process` converts this to a readable report. For memory, `v8.writeHeapSnapshot()` creates a heap snapshot that you load into Chrome DevTools Memory tab to find objects that are retained but should have been freed. The snapshot reveals the retention path — the chain of references keeping an object alive.

In production, Clinic.js provides diagnostic tools: `clinic doctor` gives an overview of event loop, CPU, memory, and handles; `clinic flame` generates a flamegraph showing CPU hotspots; `clinic bubbleprof` visualizes async flow and latency. For continuous monitoring, you integrate heap snapshot endpoints behind authentication so you can capture snapshots on demand from a running service.

The fix for unbounded caches is to use a bounded cache library like `node-cache` with `stdTTL` (time-to-live) and `maxKeys` limits. For event listener leaks, use `once()` instead of `on()` when you only need a single event, and always clean up listeners in finally blocks. For closure leaks, avoid capturing large objects in long-lived closures — extract only the specific values you need.

**Key Points to Hit:**
- [ ] Monitor process.memoryUsage() hourly; alert on continuous heap growth
- [ ] Common leaks: globals, event listeners, closures, unbounded caches, request context
- [ ] v8.writeHeapSnapshot() + Chrome DevTools for finding retention paths
- [ ] Clinic.js for production diagnostics: doctor, flame, bubbleprof
- [ ] Fix: bounded caches (node-cache with TTL), once() vs on(), minimize closure captures

**Follow-Up Questions:**
1. How do you take a heap snapshot without restarting a production container?
2. What is the difference between heapUsed and heapTotal, and why might both grow?
3. How do you distinguish between a memory leak and legitimate memory growth?

**Source:** `banking-genai-engineering-academy/backend-engineering/typescript/performance.md`

---

### Q20: How do you design a test strategy for a GenAI streaming chat interface that covers unit, integration, and E2E levels?

**Strong Answer:**

A streaming chat interface is one of the most complex components in a GenAI frontend, combining real-time data, user interaction, error handling, accessibility, and security. The test strategy must cover all of these dimensions across the testing pyramid.

At the unit level, Jest tests cover pure functions and custom hooks. The `sanitizeAndRenderMarkdown` function gets thorough tests: it renders safe markdown, strips script tags, removes event handlers, blocks javascript: URIs, removes iframes, and preserves allowed formatting. The `useChatStream` hook gets tested with mocked ReadableStreams that simulate SSE token delivery, streaming errors, and user cancellation. Tests verify that messages are added to state, streaming state transitions correctly, and errors are captured.

At the integration level, React Testing Library tests verify component behavior. The `ChatInput` component tests that it renders a textbox with proper accessibility labels, sends messages on button click and Enter key, does not send empty messages, shows a stop button during streaming, and creates new lines on Shift+Enter. The `MessageList` tests verify auto-scrolling, welcome message display, streaming indicator visibility, and error state rendering.

At the E2E level, Playwright tests cover the complete user journey: authenticate, navigate to chat, send a message, verify the user message appears, verify the assistant response streams, verify the stop button works, verify feedback submission, and verify the chat page is accessible (skip navigation, proper labels, headings). The tests wait for specific conditions rather than using arbitrary timeouts to avoid flakiness.

For streaming specifically, the unit tests mock the ReadableStream to emit tokens with controlled timing. The integration tests verify that the component renders partial content correctly during streaming. The E2E tests verify the end-to-end flow from user input to complete streamed response, including the "stop generating" interaction.

Coverage thresholds for a banking application are aggressive: 80% line coverage, 80% function coverage, 75% branch coverage. The CI pipeline runs unit tests on every PR, integration tests on every PR, and E2E tests on every PR against a staging deployment. Visual regression tests capture screenshots of the chat interface and compliance dashboard, flagging any unexpected visual changes.

**Key Points to Hit:**
- [ ] Unit: sanitizeAndRenderMarkdown tests (XSS prevention), useChatStream with mock streams
- [ ] Integration: ChatInput behavior, MessageList rendering, accessibility labels
- [ ] E2E: full user journey with Playwright — send, stream, stop, feedback, a11y
- [ ] Streaming tests: mock ReadableStream with controlled timing, verify partial renders
- [ ] Banking thresholds: 80% lines, 80% functions, 75% branches; CI on every PR

**Follow-Up Questions:**
1. How do you test that the abort signal actually cancels the fetch request?
2. How do you write a test for a race condition between two concurrent messages?
3. How do you handle flaky E2E tests in CI?

**Source:** `banking-genai-engineering-academy/frontend-engineering/frontend-testing.md`, `banking-genai-engineering-academy/frontend-engineering/streaming-responses.md`
