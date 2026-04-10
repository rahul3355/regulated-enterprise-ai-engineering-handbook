# Frontend Engineering Terms

> Essential frontend engineering terminology for GenAI platform engineers. Each term includes definition, banking/GenAI example, related concepts, and common misunderstandings.

## Glossary

### 1. SPA (Single Page Application)
**Definition:** A web application that loads a single HTML page and dynamically updates content using JavaScript, without full page reloads.
**Example:** The GenAI assistant web interface is an SPA — when users send queries, only the chat area updates, not the entire page.
**Related Concepts:** React, routing, client-side rendering, SSR
**Common Misunderstanding:** SPAs are not inherently faster. The initial load may be slower (downloading all JavaScript), but subsequent interactions are faster (no page reload).

### 2. SSR (Server-Side Rendering)
**Definition:** Rendering HTML on the server and sending the fully rendered page to the browser, improving initial load time and SEO.
**Example:** The GenAI platform's documentation portal uses SSR so that search engines can index the content and users see content immediately.
**Related Concepts:** CSR, SSG, hydration, Next.js, SEO
**Common Misunderstanding:** SSR doesn't eliminate JavaScript. The browser still downloads JS to make the page interactive (hydration).

### 3. Component
**Definition:** A reusable, self-contained piece of UI that encapsulates structure, styling, and behavior.
**Example:** The GenAI assistant has components: `ChatMessage`, `CitationCard`, `TypingIndicator`, `QueryInput`, `SafetyWarning` — each independently developed and tested.
**Related Concepts:** Props, state, composition, design system, Storybook
**Common Misunderstanding:** Components are not just visual. They can be purely logical (a data-fetching component) or purely behavioral (an analytics tracking component).

### 4. Props
**Definition:** Read-only data passed from a parent component to a child component to configure its behavior or appearance.
**Example:** The `ChatMessage` component receives `message` (text), `sender` (user/assistant), `timestamp`, and `citations` as props from its parent.
**Related Concepts:** State, unidirectional data flow, children
**Common Misunderstanding:** Props are read-only. A child component cannot modify props received from its parent. If it needs to change data, it should emit an event or callback.

### 5. State
**Definition:** Data that changes over time within a component or application, triggering UI updates when modified.
**Example:** The GenAI chat interface maintains state: the list of messages, the current input text, loading status, and error state.
**Related Concepts:** Props, useState, state management, immutability
**Common Misunderstanding:** State is not the same as props. Props come from the parent and are read-only. State is owned by the component and can change.

### 6. Hooks
**Definition:** Functions that let you use React features (state, lifecycle, context) in functional components without writing classes.
**Example:** `useState` manages the chat input text, `useEffect` fetches the response when a query is submitted, `useContext` accesses the user's authentication state.
**Related Concepts:** useState, useEffect, useContext, custom hooks
**Common Misunderstanding:** Hooks must be called in the same order every render. They cannot be called conditionally or inside loops.

### 7. Virtual DOM
**Definition:** A lightweight JavaScript representation of the actual DOM that React uses to batch and optimize UI updates.
**Example:** When a new chat message arrives, React updates the virtual DOM, compares it with the previous version, and only updates the changed parts of the actual DOM.
**Related Concepts:** Reconciliation, diffing, rendering, performance
**Common Misunderstanding:** The virtual DOM is not inherently faster than direct DOM manipulation. It's a trade-off: slightly slower for simple updates, much faster for complex updates.

### 8. Client-Side Routing
**Definition:** Navigation within an SPA that updates the URL and content without reloading the page, handled by JavaScript.
**Example:** The GenAI platform uses routes: `/chat` (main chat), `/history` (query history), `/settings` (preferences), `/admin` (admin panel) — all handled client-side.
**Related Concepts:** BrowserRouter, Route, Link, navigation guards
**Common Misunderstanding:** Client-side routing changes the URL without a server request. But if the user refreshes the page, the server must be configured to serve the SPA for all routes.

### 9. Accessibility (a11y)
**Definition:** Designing and building applications that can be used by people with disabilities, including visual, motor, auditory, and cognitive impairments.
**Example:** The GenAI chat interface uses semantic HTML, ARIA labels, keyboard navigation, proper color contrast, and screen reader support so that all employees can use it.
**Related Concepts:** WCAG, ARIA, screen reader, keyboard navigation, color contrast
**Common Misunderstanding:** Accessibility is not just for blind users. It includes keyboard-only users (motor impairments), color-blind users, users with cognitive differences, and temporary impairments (broken arm).

### 10. ARIA (Accessible Rich Internet Applications)
**Definition:** A set of attributes that add semantic meaning to web content for assistive technologies like screen readers.
**Example:** The GenAI typing indicator uses `aria-live="polite"` so screen readers announce "Assistant is typing..." when the status changes.
**Related Concepts:** Accessibility, semantic HTML, screen reader
**Common Misunderstanding:** ARIA is not a substitute for semantic HTML. Use proper HTML elements first (`<button>`, `<nav>`, `<main>`), then add ARIA for additional meaning.

### 11. Web Vitals
**Definition:** A set of metrics that measure real-world user experience: LCP (loading), FID (interactivity), CLS (visual stability).
**Example:** The GenAI assistant's LCP is 1.2s (good: <2.5s), FID is 50ms (good: <100ms), and CLS is 0.05 (good: <0.1) — all within Google's "good" thresholds.
**Related Concepts:** Performance, Core Web Vitals, LCP, FID, CLS, INP
**Common Misunderstanding:** Web Vitals measure perceived user experience, not raw technical performance. A page can have fast server response but poor Web Vitals due to large images or layout shifts.

### 12. Design System
**Definition:** A collection of reusable components, design tokens, and guidelines that ensure consistent visual and interaction patterns across an application.
**Example:** The bank's design system provides: button styles, form controls, color palette, typography, spacing tokens, and GenAI-specific components (chat bubbles, citation cards, confidence indicators).
**Related Concepts:** Component library, design tokens, Storybook, brand guidelines
**Common Misunderstanding:** A design system is not just a component library. It includes design principles, usage guidelines, accessibility standards, and contribution processes.

### 13. Responsive Design
**Definition:** Designing web interfaces that adapt to different screen sizes and devices (desktop, tablet, mobile).
**Example:** The GenAI assistant chat interface works on desktop (full sidebar + chat), tablet (collapsible sidebar), and mobile (full-screen chat with bottom input bar).
**Related Concepts:** Breakpoints, media queries, flexbox, grid, mobile-first
**Common Misunderstanding:** Responsive design is not just about making things smaller on mobile. It's about rethinking the layout and interaction model for different contexts of use.

### 14. XSS (Cross-Site Scripting)
**Definition:** An attack where malicious scripts are injected into a web application and executed in other users' browsers.
**Example:** If the GenAI assistant displays user queries without escaping, an attacker could submit `<img src=x onerror="alert(document.cookie)">` and execute JavaScript in other users' browsers.
**Related Concepts:** Input sanitization, output encoding, CSP, DOMPurify
**Common Misunderstanding:** Modern frameworks (React, Angular) auto-escape by default, but XSS is still possible through `dangerouslySetInnerHTML` (React), `v-html` (Vue), or `innerHTML` (vanilla JS).

### 15. CSP (Content Security Policy)
**Definition:** An HTTP header that restricts which resources (scripts, styles, images) the browser can load, preventing XSS and data injection attacks.
**Example:** The GenAI assistant's CSP only allows scripts from the bank's own domain and trusted CDNs, blocking any inline scripts or scripts from unknown sources.
**Related Concepts:** XSS, nonce, trusted types, frame-ancestors
**Common Misunderstanding:** CSP is not a silver bullet. It reduces the impact of XSS but doesn't prevent it. You still need input validation and output encoding.

### 16. CSRF Token
**Definition:** A unique, secret token included in forms and requests to verify that the request originated from the legitimate application, not a malicious site.
**Example:** The GenAI admin panel includes a CSRF token in every form submission. If an attacker tries to forge a request from another site, it fails without the token.
**Related Concepts:** SameSite cookies, CORS, authentication, session
**Common Misunderstanding:** CSRF tokens are not needed for APIs that use Authorization headers (Bearer tokens). They're needed for cookie-based authentication.

### 17. WebSocket
**Definition:** A protocol providing full-duplex, bidirectional communication between a client and server over a single persistent connection.
**Example:** The GenAI assistant uses WebSocket for real-time streaming responses — tokens are sent from server to client as they're generated, providing a ChatGPT-like experience.
**Related Concepts:** SSE, long polling, Server-Sent Events, real-time
**Common Misunderstanding:** WebSocket is not always better than SSE. If you only need server-to-client streaming (not client-to-server), SSE is simpler and works over HTTP.

### 18. SSE (Server-Sent Events)
**Definition:** A standard allowing a server to push real-time updates to a browser client over a single HTTP connection.
**Example:** The GenAI chat API streams token-by-token responses using SSE. The client receives each token as an event and progressively displays the response.
**Related Concepts:** WebSocket, streaming, EventSource, chunked transfer
**Common Misunderstanding:** SSE is unidirectional (server to client only). For bidirectional communication, use WebSocket.

### 19. Progressive Enhancement
**Definition:** A design strategy where the core content and functionality work without JavaScript, and JavaScript adds enhanced features on top.
**Example:** The GenAI documentation portal works as a basic HTML site without JavaScript. JavaScript adds: search autocomplete, interactive code examples, and dark mode toggle.
**Related Concepts:** Graceful degradation, SPA, SSR, accessibility
**Common Misunderstanding:** Progressive enhancement is not "make it work in IE6." It's about ensuring the core experience works even when advanced features fail.

### 20. Bundle Size
**Definition:** The total size of JavaScript, CSS, and other assets that must be downloaded and parsed by the browser before the application becomes interactive.
**Example:** The GenAI assistant's initial bundle is 350KB (gzipped). Code splitting reduces this to 150KB for the initial chat view, with other modules loaded on demand.
**Related Concepts:** Code splitting, tree shaking, lazy loading, webpack, Vite
**Common Misunderstanding:** Bundle size is not just about download time. Large bundles also take longer to parse and compile in the browser, delaying time to interactive.

### 21. Code Splitting
**Definition:** Dividing the application's JavaScript bundle into smaller chunks that are loaded on demand, reducing initial load time.
**Example:** The GenAI admin panel is code-split — it's only loaded when a user with admin privileges navigates to `/admin`, not on initial page load.
**Related Concepts:** Lazy loading, dynamic import, bundle size, route-based splitting
**Common Misunderstanding:** Code splitting adds complexity: you need loading states for each chunk, and too many small chunks can hurt performance due to HTTP overhead.

### 22. State Management
**Definition:** The patterns and tools for managing application-wide state that multiple components need to access or modify.
**Example:** The GenAI assistant uses React Context for: user authentication state, theme preferences, and feature flags. For complex state (chat history), it uses a custom reducer pattern.
**Related Concepts:** Redux, Zustand, Context API, useReducer, store
**Common Misunderstanding:** You don't need Redux for every app. React's built-in Context and useReducer handle most use cases. Add external state management only when you need it.

### 23. Hydration
**Definition:** The process of attaching JavaScript event handlers to server-rendered HTML, making a static page interactive.
**Example:** The GenAI documentation portal is server-rendered for fast initial display. When JavaScript loads, it "hydrates" the page — adding event handlers for search, navigation, and code copy buttons.
**Related Concepts:** SSR, CSR, partial hydration, islands architecture
**Common Misunderstanding:** Hydration is expensive — the browser must download, parse, and execute all JavaScript even though the HTML is already displayed. This is why TTI (Time to Interactive) can be much later than FCP (First Contentful Paint).

### 24. Service Worker
**Definition:** A script that runs in the browser background, independent of the web page, enabling features like offline access and push notifications.
**Example:** The GenAI assistant's service worker caches the application shell and recent chat responses, allowing the app to load and display cached conversations even when offline.
**Related Concepts:** PWA, offline, cache, push notifications, Workbox
**Common Misunderstanding:** Service workers are powerful but complex. They intercept all network requests, which means bugs in a service worker can break the entire application.

### 25. Lighthouse
**Definition:** An automated tool by Google that audits web pages for performance, accessibility, best practices, and SEO.
**Example:** Before deploying the GenAI assistant, the team runs Lighthouse CI and requires all scores to be above 90. This catches performance regressions and accessibility issues before they reach users.
**Related Concepts:** Web Vitals, performance audit, CI/CD, synthetic monitoring
**Common Misunderstanding:** Lighthouse scores are lab measurements (controlled environment), not real user measurements. They're useful for catching regressions but don't replace real user monitoring (RUM).
