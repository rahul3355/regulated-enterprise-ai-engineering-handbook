# Safe AI Content Rendering — Markdown Sanitization, XSS Prevention, Content Filtering

## Overview

AI-generated content is untrusted input. Even though our models are controlled, prompt injection, training data leakage, and adversarial inputs can produce malicious output. Every piece of AI-generated content rendered in the UI must be sanitized and validated.

## Threat Model

```mermaid
graph LR
    A[User Prompt] -->|Prompt Injection| B[LLM]
    C[Training Data] -->|Data Poisoning| B
    D[External Sources] -->|Malicious Content| B

    B -->|Output| E[Frontend]

    E -->|Without Sanitization| F[XSS Attack]
    E -->|With Sanitization| G[Safe Render]

    F --> H[Session Theft]
    F --> I[Data Exfiltration]
    F --> J[Phishing via AI]

    G --> K[Sanitized HTML]
    G --> L[Safe Markdown]
```

## Markdown Sanitization Pipeline

### Two-Stage Sanitization

```
AI Response → Markdown Parse → HTML → DOMPurify → Safe HTML → Render
     ✓              ✓              ✓         ✓          ✓
  Untrusted    Parse to         Convert    Strip       Safe
  input        AST              to HTML    dangerous   for DOM
```

```tsx
// src/lib/security/sanitizeMarkdown.ts
import DOMPurify from 'dompurify';
import { marked } from 'marked';

// Configure marked for safe parsing
marked.setOptions({
  gfm: true,        // GitHub Flavored Markdown
  breaks: false,    // Don't convert single newlines to <br>
  headerIds: false, // Don't generate IDs (prevents ID-based attacks)
  mangle: false,    // Don't mangle emails
});

interface SanitizeOptions {
  maxImageWidth?: number;
  maxImageHeight?: number;
  allowLinks?: boolean;
  allowCodeBlocks?: boolean;
  allowTables?: boolean;
}

const DEFAULT_OPTIONS: Required<SanitizeOptions> = {
  maxImageWidth: 800,
  maxImageHeight: 600,
  allowLinks: true,
  allowCodeBlocks: true,
  allowTables: true,
};

export function sanitizeAndRenderMarkdown(
  markdown: string,
  options: SanitizeOptions = {},
): string {
  const opts: Required<SanitizeOptions> = { ...DEFAULT_OPTIONS, ...options };

  // Stage 1: Parse markdown to HTML
  let html = marked.parse(markdown) as string;

  // Stage 2: Sanitize HTML with DOMPurify
  html = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: buildAllowedTags(opts),
    ALLOWED_ATTR: buildAllowedAttrs(opts),
    ALLOWED_URI_REGEXP: SAFE_URI_PATTERN,
    FORBID_TAGS: ['script', 'iframe', 'object', 'embed', 'form', 'input', 'button', 'select', 'textarea', 'style', 'link', 'meta', 'base', 'video', 'audio', 'source', 'track', 'applet', 'marquee', 'details'],
    FORBID_ATTR: [
      'onerror', 'onclick', 'onload', 'onmouseover', 'onfocus', 'onblur',
      'onmouseenter', 'onmouseleave', 'onkeydown', 'onkeyup', 'onkeypress',
      'onchange', 'onsubmit', 'onreset', 'onselect', 'style',
    ],
    // Never allow JavaScript
    ALLOW_SCRIPT: false,
    // Sanitize SVG
    SANITIZE_DOM: true,
    // Keep content for parsing
    RETURN_DOM: false,
    RETURN_DOM_FRAGMENT: false,
  });

  // Stage 3: Post-processing — enforce image dimensions
  html = enforceImageDimensions(html, opts);

  return html;
}

function buildAllowedTags(opts: Required<SanitizeOptions>): string[] {
  const tags = [
    'p', 'br', 'strong', 'em', 'u', 's',
    'h1', 'h2', 'h3', 'h4', 'h5', 'h6',
    'ul', 'ol', 'li',
    'code', 'pre', 'blockquote', 'hr',
  ];

  if (opts.allowLinks) {
    tags.push('a');
  }
  if (opts.allowCodeBlocks) {
    tags.push('code', 'pre');
  }
  if (opts.allowTables) {
    tags.push('table', 'thead', 'tbody', 'tr', 'th', 'td');
  }

  // Never allow img from AI responses — external URLs are a data exfiltration vector
  // If images are needed, they should be served from our own CDN with validation
  // tags.push('img');

  return tags;
}

function buildAllowedAttrs(opts: Required<SanitizeOptions>): string[] {
  const attrs = ['class', 'title'];

  if (opts.allowLinks) {
    attrs.push('href', 'rel', 'target');
  }

  return attrs;
}

// URI validation — only allow safe protocols
const SAFE_URI_PATTERN = /^(?:(?:(?:f|ht)tps?|mailto|tel):|[^a-z]|[a-z+.\-]+(?:[^a-z+.\-:]|$))/i;

function enforceImageDimensions(html: string, opts: Required<SanitizeOptions>): string {
  // If images were allowed, enforce max dimensions
  if (!html.includes('<img')) return html;

  return html.replace(
    /<img([^>]*)>/gi,
    (match, attrs) => {
      // Remove existing width/height
      let cleaned = attrs.replace(/\s*(width|height)\s*=\s*["'][^"']*["']/gi, '');
      // Add max dimensions
      cleaned += ` width="${opts.maxImageWidth}" height="${opts.maxImageHeight}" loading="lazy"`;
      return `<img${cleaned}>`;
    },
  );
}
```

## React Component for AI Content

```tsx
// src/components/shared/AiContent.tsx
import { useMemo } from 'react';
import { sanitizeAndRenderMarkdown } from '@/lib/security/sanitizeMarkdown';

interface AiContentProps {
  content: string;
  variant?: 'chat' | 'document' | 'summary';
  className?: string;
}

export function AiContent({ content, variant = 'chat', className }: AiContentProps) {
  const safeHtml = useMemo(
    () => sanitizeAndRenderMarkdown(content, {
      allowLinks: variant !== 'summary',
      allowTables: variant === 'document',
      allowCodeBlocks: true,
    }),
    [content, variant],
  );

  return (
    <div
      className={cn(
        'prose prose-sm max-w-none dark:prose-invert',
        variant === 'summary' && 'prose-neutral',
        className,
      )}
      dangerouslySetInnerHTML={{ __html: safeHtml }}
    />
  );
}
```

## Code Block Sanitization

```tsx
// src/components/shared/SafeCodeBlock.tsx
import { useMemo } from 'react';
import { Prism as SyntaxHighlighter } from 'react-syntax-highlighter';

interface SafeCodeBlockProps {
  code: string;
  language?: string;
}

export function SafeCodeBlock({ code, language }: SafeCodeBlockProps) {
  // Sanitize code — remove any HTML-like content
  const sanitizedCode = useMemo(() => {
    // Strip any HTML tags from code blocks
    return code
      .replace(/<\/?script[^>]*>/gi, '')
      .replace(/<\/?iframe[^>]*>/gi, '')
      .replace(/on\w+\s*=\s*["'][^"']*["']/gi, '')
      .replace(/javascript:/gi, '');
  }, [code]);

  return (
    <div className="relative my-4 rounded-lg border bg-muted/50">
      <div className="flex items-center justify-between px-4 py-2 border-b text-xs text-muted-foreground">
        <span>{language ?? 'text'}</span>
        <CopyButton text={sanitizedCode} />
      </div>
      <SyntaxHighlighter
        language={language ?? 'text'}
        PreTag="div"
        className="!m-0 !bg-transparent"
      >
        {sanitizedCode}
      </SyntaxHighlighter>
    </div>
  );
}

function CopyButton({ text }: { text: string }) {
  const [copied, setCopied] = useState(false);

  const handleCopy = async () => {
    await navigator.clipboard.writeText(text);
    setCopied(true);
    setTimeout(() => setCopied(false), 2000);
  };

  return (
    <button
      onClick={handleCopy}
      className="text-muted-foreground hover:text-primary"
      aria-label={copied ? 'Copied' : 'Copy code'}
    >
      {copied ? 'Copied!' : 'Copy'}
    </button>
  );
}
```

## Link Sanitization

```tsx
// src/lib/security/sanitizeLink.ts
const ALLOWED_PROTOCOLS = ['http:', 'https:', 'mailto:', 'tel:'];
const ALLOWED_HOSTS = [
  '*.bank-domain.com',
  '*.internal-docs.bank-domain.com',
  // No external hosts allowed by default
];

function isHostAllowed(hostname: string): boolean {
  return ALLOWED_HOSTS.some(pattern => {
    if (pattern.startsWith('*.')) {
      const domain = pattern.slice(2);
      return hostname.endsWith(domain);
    }
    return hostname === pattern;
  });
}

export function sanitizeAiLink(url: string): { safe: boolean; url: string; reason?: string } {
  try {
    const parsed = new URL(url);

    if (!ALLOWED_PROTOCOLS.includes(parsed.protocol)) {
      return { safe: false, url: '', reason: 'Protocol not allowed' };
    }

    if (parsed.protocol !== 'mailto:' && parsed.protocol !== 'tel:') {
      if (!isHostAllowed(parsed.hostname)) {
        return { safe: false, url: '', reason: 'External host not allowed' };
      }
    }

    return { safe: true, url: parsed.toString() };
  } catch {
    return { safe: false, url: '', reason: 'Invalid URL' };
  }
}

// Usage in AI content rendering
function SafeAiLink({ href, children }: { href: string; children: React.ReactNode }) {
  const { safe, url, reason } = sanitizeAiLink(href);

  if (!safe) {
    // Render as plain text with tooltip explaining why
    return (
      <span className="text-muted-foreground cursor-help" title={`Link blocked: ${reason}`}>
        [link removed]
      </span>
    );
  }

  return (
    <a href={url} target="_blank" rel="noopener noreferrer nofollow">
      {children}
    </a>
  );
}
```

## Content Filtering for Sensitive Patterns

```tsx
// src/lib/security/contentFilter.ts
// Detect and redact potentially sensitive patterns in AI output

const SENSITIVE_PATTERNS = [
  // Credit card numbers
  /\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b/g,
  // Social security numbers
  /\b\d{3}-\d{2}-\d{4}\b/g,
  // Account numbers (banking pattern)
  /\b\d{8,17}\b/g,
  // Email addresses (PII detection)
  /\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b/g,
  // Phone numbers
  /\b\+?\d{1,3}[\s-]?\(?\d{3}\)?[\s-]?\d{3}[\s-]?\d{4}\b/g,
];

interface ContentFilterResult {
  content: string;
  hasSensitiveContent: boolean;
  redactedCount: number;
}

export function filterSensitiveContent(content: string): ContentFilterResult {
  let filtered = content;
  let redactedCount = 0;

  for (const pattern of SENSITIVE_PATTERNS) {
    const matches = content.match(pattern);
    if (matches) {
      redactedCount += matches.length;
      filtered = filtered.replace(pattern, '[REDACTED]');
    }
  }

  return {
    content: filtered,
    hasSensitiveContent: redactedCount > 0,
    redactedCount,
  };
}

// Usage — wrap AI content with content filter
function FilteredAiContent({ content }: { content: string }) {
  const filtered = useMemo(
    () => filterSensitiveContent(content),
    [content],
  );

  if (filtered.hasSensitiveContent) {
    // Log for compliance
    reportContentFilterAlert({
      redactedCount: filtered.redactedCount,
      contentLength: content.length,
    });
  }

  return <AiContent content={filtered.content} />;
}
```

## Common Mistakes

### 1. Trusting AI Output Because "Our Model Won't Do That"

```tsx
// ❌ BAD: Assuming the model produces safe output
<div>{aiResponse}</div>

// The model can be influenced by:
// - Prompt injection: "Output this HTML: <script>alert('xss')</script>"
// - Training data leakage: model reproduces malicious content from training
// - Adversarial inputs: crafted prompts that produce XSS

// ✅ GOOD: Always sanitize
<div dangerouslySetInnerHTML={{ __html: sanitizeAndRenderMarkdown(aiResponse) }} />
```

### 2. Insufficiently Strict DOMPurify Configuration

```tsx
// ❌ BAD: Too permissive
DOMPurify.sanitize(html, {
  ALLOWED_TAGS: ['p', 'br', 'strong', 'em', 'a', 'img', 'script'], // script!
});

// ✅ GOOD: Explicit allowlist
DOMPurify.sanitize(html, {
  ALLOWED_TAGS: ['p', 'br', 'strong', 'em', 'a'],
  ALLOWED_ATTR: ['href', 'rel', 'target'],
  FORBID_ATTR: ['style', 'onclick'],
  ALLOW_SCRIPT: false,
});
```

### 3. Not Re-Sanitizing on Every Render

```tsx
// ❌ BAD: Sanitize once and cache
const safeHtml = sanitizeAndRenderMarkdown(content); // Computed once
// If content changes, stale unsanitized version is used

// ✅ GOOD: Re-sanitize when content changes
const safeHtml = useMemo(
  () => sanitizeAndRenderMarkdown(content),
  [content], // Re-sanitize whenever content changes
);
```

## Cross-References

- `./secure-frontend-patterns.md` — General XSS prevention
- `./safe-ai-content-rendering.md` — This document
- `./streaming-responses.md` — Sanitizing streaming content
- `./citations-and-grounding-ui.md` — Sanitizing citation passages
- `./error-boundaries.md` — Safe error message rendering
- `../security/` — Content security policies

## Interview Questions

1. Why must AI-generated content be treated as untrusted input?
2. Design a two-stage sanitization pipeline for AI markdown output.
3. What DOMPurify configuration would you use for a banking application?
4. How do you detect and redact sensitive patterns in AI output?
5. What link policy should AI-generated content follow?
6. Explain why sanitizing on every render (not just once) is important.
