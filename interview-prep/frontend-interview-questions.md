# Frontend Interview Questions (30+)

## Foundational Frontend

### Q1: Explain the critical rendering path in a browser.

**Strong Answer**: "The critical rendering path is the sequence of steps the browser takes to convert HTML, CSS, and JavaScript into pixels on screen: (1) **Parse HTML** to build the DOM tree. (2) **Parse CSS** to build the CSSOM tree. (3) **Combine DOM + CSSOM** into the render tree (excludes hidden elements). (4) **Layout** -- calculate the position and size of each element. (5) **Paint** -- fill in pixels for each element. (6) **Composite** -- layer elements and composite them together. JavaScript blocks parsing (it's a render-blocking resource), which is why we put scripts at the bottom or use `defer`/`async`. CSS blocks rendering (but not parsing), which is why above-the-fold CSS should be inlined. Optimization strategies: minimize render-blocking resources, use `preload` for critical assets, defer non-critical JavaScript, and optimize CSS delivery."

### Q2: What is the virtual DOM and why does React use it?

**Strong Answer**: "The virtual DOM is a lightweight JavaScript representation of the real DOM. When state changes, React: (1) Creates a new virtual DOM tree reflecting the updated state. (2) Diffs it against the previous virtual DOM tree (reconciliation). (3) Computes the minimal set of changes needed. (4) Batches and applies those changes to the real DOM. The real DOM is slow to manipulate because each change can trigger layout recalculation and repainting. The virtual DOM lets React batch updates and minimize expensive DOM operations. However, the virtual DOM isn't inherently faster than direct DOM manipulation -- it's a tradeoff that enables declarative programming and consistent performance regardless of application complexity."

### Q3: Explain the difference between `let`, `const`, and `var`.

**Strong Answer**: "`var` is function-scoped and hoisted (initialized as `undefined` at the top of the function). `let` and `const` are block-scoped (limited to the nearest `{}`) and not hoisted in the same way (they're in the temporal dead zone until declared). `const` prevents reassignment of the variable binding (but the value itself can be mutated if it's an object). Best practice: use `const` by default, `let` when you need reassignment, and never use `var` in modern code. `var` exists for backward compatibility and has confusing behavior (like being accessible before its declaration due to hoisting)."

## React-Specific

### Q4: Explain React's component lifecycle and the equivalent hooks.

**Strong Answer**: "Class component lifecycle: **mounting** (constructor -> render -> componentDidMount), **updating** (render -> componentDidUpdate), **unmounting** (componentWillUnmount). With hooks: `useState` handles state across renders, `useEffect` replaces lifecycle methods (empty dependency array = componentDidMount, dependency array = componentDidUpdate, return function = componentWillUnmount), `useMemo` memoizes expensive computations, `useCallback` memoizes functions. Key principle: effects run after paint, so they don't block rendering. Cleanup functions in useEffect run before the next effect and on unmount, preventing memory leaks from subscriptions or event listeners."

### Q5: How do you optimize React performance for a large dashboard?

**Strong Answer**: "Multiple strategies: (1) **Memoization**: `React.memo` for components that render often with same props, `useMemo` for expensive computations, `useCallback` for functions passed to memoized children. (2) **Code splitting**: `React.lazy` + `Suspense` to load dashboard sections on demand. (3) **Virtualization**: For long lists, use `react-window` to render only visible rows. (4) **Avoid unnecessary re-renders**: Proper dependency arrays in `useEffect`, avoid inline object/array creation in render, use state colocation (keep state close to where it's used). (5) **Debounce expensive operations**: Search input debouncing, scroll event throttling. (6) **Server-side data fetching**: Use React Query or SWR for caching, deduplication, and background refetching. (7) **Profile first**: Use React DevTools Profiler to identify actual bottlenecks before optimizing."

### Q6: What is the difference between controlled and uncontrolled components?

**Strong Answer**: "Controlled components: React manages the form state via `value` prop and `onChange` handler. The source of truth is React state. Uncontrolled components: the DOM manages the state, and React reads values via `ref`. Controlled is preferred because: state is predictable, validation is easy, conditional disabling works naturally, and instant feedback is possible. Uncontrolled is useful for: integrating with non-React libraries, file inputs (which can't be controlled), and very large forms where controlled state updates cause performance issues."

## CSS and Layout

### Q7: Explain CSS Grid vs. Flexbox. When do you use each?

**Strong Answer**: "Flexbox is one-dimensional (row OR column) and content-driven (items determine layout). Grid is two-dimensional (rows AND columns) and layout-driven (container determines layout). Use Flexbox for: navigation bars, card rows, centering content, flexible item distribution. Use Grid for: page layouts, complex multi-row/column arrangements, overlapping elements, when you need explicit placement. In practice, I often use Grid for the overall page structure and Flexbox for component-level layouts. Grid has better support for responsive design with `minmax()` and `auto-fill`/`auto-fit`."

### Q8: How do you handle responsive design?

**Strong Answer**: "Mobile-first approach: (1) Design for mobile first, then add breakpoints for larger screens. (2) Use relative units (`rem`, `em`, `%`, `vw/vh`) instead of fixed pixels. (3) CSS media queries for layout changes: `@media (min-width: 768px) { ... }`. (4) Responsive images with `srcset` and `sizes` attributes. (5) Fluid typography with `clamp()` for smooth scaling. (6) Container queries for component-level responsiveness (newer but powerful). (7) Test on real devices, not just browser dev tools. For banking applications, accessibility is also critical -- responsive design must maintain WCAG compliance at all screen sizes."

## JavaScript Deep Dive

### Q9: Explain closures with a practical example.

**Strong Answer**: "A closure is a function that has access to variables from its outer (enclosing) lexical scope, even after the outer function has returned. Practical example: a counter factory:
```javascript
function createCounter() {
    let count = 0; // This variable is captured by the closure
    return {
        increment: () => ++count,
        getCount: () => count
    };
}
const counter = createCounter();
counter.increment(); // 1
counter.getCount();  // 1
// count is not accessible from outside, but the closure keeps it alive
```
Common use cases: data privacy (encapsulation), function factories, event handlers that need access to scope, and useCallback/useMemo dependencies in React. Memory caution: closures keep referenced variables in memory, so be careful with large objects in long-lived closures."

### Q10: Explain event delegation and why it's useful.

**Strong Answer**: "Event delegation attaches a single event listener to a parent element instead of individual listeners to each child. It works because events bubble up the DOM tree. When an event fires on a child, the parent's listener receives it and can check `event.target` to identify which child triggered it. Benefits: (1) **Memory efficiency** -- one listener instead of hundreds. (2) **Dynamic elements** -- works for children added after the listener is attached. (3) **Simpler cleanup** -- remove one listener instead of many. Example: a table with 1000 rows -- instead of 1000 click listeners, one listener on the `<table>` checks `event.target.closest('tr')`. In React, this is handled automatically by the synthetic event system."

## State Management

### Q11: When would you use Redux vs. Context API vs. local state?

**Strong Answer**: "**Local state** (`useState`) is the default -- use it whenever possible. **Context API** is for data that many components need but changes infrequently (theme, user auth, locale). It's built-in and simple but triggers re-renders of all consumers when the value changes. **Redux/Zustand** is for complex state logic: frequent updates, complex state transitions, time-travel debugging, middleware needs (logging, analytics), or when many components need to read and write shared state. In a banking dashboard, I'd use: local state for form inputs, Context for user authentication and theme, and Redux/Zustand for real-time data like account balances that multiple components need. The key principle: start with the simplest solution and escalate only when needed."

## Performance

### Q12: How do you identify and fix a performance bottleneck in a web app?

**Strong Answer**: "Systematic approach: (1) **Measure first**: Use Chrome DevTools Performance tab, Lighthouse, and Web Vitals (LCP, FID, CLS) to identify the actual bottleneck. Don't guess. (2) **Network bottlenecks**: Check Waterfall view. Fix: compress assets, enable HTTP/2, use CDN, lazy load images, code split. (3) **JavaScript bottlenecks**: Check Long Tasks in Performance tab. Fix: defer non-critical JS, web workers for heavy computation, optimize React re-renders. (4) **Rendering bottlenecks**: Check Layout Shifts and excessive paints. Fix: set explicit dimensions, avoid layout thrashing, use CSS transforms instead of layout-changing properties. (5) **Memory bottlenecks**: Check Memory tab for leaks. Fix: remove event listeners, clean up subscriptions in useEffect, avoid retaining references to large objects."

## Security

### Q13: How do you prevent XSS in a React application?

**Strong Answer**: "React auto-escapes all JSX content by default, which prevents most XSS attacks. But vulnerabilities still exist: (1) **`dangerouslySetInnerHTML`** -- bypasses React's escaping. Only use with sanitized HTML (DOMPurify). (2) **`href` attributes** -- `href="javascript:..."` can execute code. Validate URLs. (3) **User-controlled `src`** -- `<img src={userInput}>` can trigger onload handlers. (4) **Third-party libraries** -- any library that manipulates the DOM directly could introduce XSS. Defense: sanitize all user input before rendering, use Content Security Policy headers, avoid `dangerouslySetInnerHTML` when possible, and regularly audit dependencies."

## Banking-Specific Frontend Questions

### Q14: How do you design a banking dashboard that's accessible to all users?

**Strong Answer**: "WCAG 2.1 AA compliance: (1) **Color contrast** -- minimum 4.5:1 for normal text, 3:1 for large text. Don't use color alone to convey information (red for negative, green for positive needs icons or labels too). (2) **Keyboard navigation** -- all interactive elements reachable and operable via keyboard. Visible focus indicators. (3) **Screen reader support** -- semantic HTML, ARIA labels where needed, live regions for dynamic updates (balance changes). (4) **Form accessibility** -- labels for all inputs, error messages associated with inputs via `aria-describedby`. (5) **Responsive design** -- works at all zoom levels up to 400%. (6) **Clear language** -- avoid banking jargon, provide explanations for terms. (7) **Testing** -- automated (axe, Lighthouse) and manual (screen reader testing, keyboard-only navigation). In banking, accessibility is not just ethical -- it's often a legal requirement."

## Quick-Fire Frontend Questions

### Q15: What is the difference between `==` and `===`?
**Answer**: "`==` does type coercion before comparison (`'5' == 5` is true). `===` checks both value and type (`'5' === 5` is false). Always use `===`."

### Q16: What is a Promise?
**Answer**: "An object representing the eventual completion or failure of an asynchronous operation. States: pending, fulfilled, rejected. Use `.then()`/`.catch()` or `async/await`."

### Q17: What is the difference between `null` and `undefined`?
**Answer**: "`undefined` means a variable has been declared but not assigned a value. `null` is an intentional absence of a value, explicitly assigned."

### Q18: What is debouncing vs. throttling?
**Answer**: "Debouncing: execute after a pause in events (search input -- wait until user stops typing). Throttling: execute at most once per interval (scroll event -- max once per 100ms)."

### Q19: What is the Same-Origin Policy?
**Answer**: "A browser security restriction that prevents a web page from making requests to a different domain, protocol, or port. CORS headers relax this restriction."

### Q20: What is SSR vs. CSR?
**Answer**: "Server-Side Rendering: HTML is generated on the server and sent to the browser (faster initial load, better SEO). Client-Side Rendering: browser downloads JS and renders the page (better interactivity, slower initial load)."

### Q21: What are Web Vitals?
**Answer**: "Google's metrics for user experience: LCP (Largest Contentful Paint -- loading speed), INP (Interaction to Next Paint -- responsiveness), CLS (Cumulative Layout Shift -- visual stability)."

### Q22: What is a service worker?
**Answer**: "A script that runs in the background, separate from the web page. Used for: offline support (caching), push notifications, background sync. Essential for PWA functionality."

### Q23: What is the difference between `localStorage` and `sessionStorage`?
**Answer**: "`localStorage` persists across browser sessions. `sessionStorage` is cleared when the tab is closed. Both are limited to ~5MB and are synchronous."

### Q24: What is CORS?
**Answer**: "Cross-Origin Resource Sharing. A mechanism that allows servers to specify which origins can access their resources via HTTP headers (Access-Control-Allow-Origin)."

### Q25: What is tree shaking?
**Answer**: "A build optimization that eliminates unused JavaScript code from the final bundle. Requires ES6 module syntax (`import`/`export`) so the bundler can statically analyze dependencies."

### Q26: What is a webpack loader vs. plugin?
**Answer**: "Loaders transform individual files (TypeScript to JS, SCSS to CSS). Plugins perform broader tasks (bundle optimization, asset management, environment variable injection)."

### Q27: What is the event loop?
**Answer**: "The mechanism that handles async operations in JavaScript. It continuously checks the call stack and, when empty, processes the next task from the callback queue. Microtasks (Promises) have priority over macrotasks (setTimeout)."

### Q28: What is React Fiber?
**Answer**: "React's reconciliation algorithm rewrite that enables incremental rendering. It can pause, abort, or reuse work, prioritize updates, and improve perceived performance by splitting rendering work into chunks."

### Q29: How do you handle API errors in a frontend app?
**Answer**: "Centralized error handling: intercept all API responses, categorize errors (network, auth, validation, server), show user-friendly messages, log errors for debugging, and provide recovery options (retry, refresh)."

### Q30: What is optimistic UI?
**Answer**: "Updating the UI immediately before the server confirms the action, assuming it will succeed. If the server fails, revert the change and show an error. Feels faster to users but requires careful rollback handling."
