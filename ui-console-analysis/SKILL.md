---
name: ui-console-analysis
description: >
  Use this skill whenever a user uploads or pastes browser console output, network
  logs, or HAR files and wants them analyzed for frontend performance issues, errors,
  or API problems. Triggers include: any mention of 'console log', 'browser console',
  'network tab', 'HAR file', 'slow page', 'slow API call', 'frontend error', 'CORS
  error', 'JavaScript error', 'failed request', '4xx', '5xx', 'network waterfall',
  'LCP', 'CLS', 'INP', 'Core Web Vitals', 'React performance', 'slow render',
  'memory leak browser', 'performance tab', 'Lighthouse', or when the user shares
  console.log output and wants to know what is wrong. Also trigger when the user asks
  why a web page is slow, why an API call fails, or what is causing browser errors.
  Always use this skill for structured, comprehensive frontend diagnosis.
---

# UI Console & Network Log Analysis Skill

## What this skill does

Analyzes browser console output, network logs, HAR files, and Lighthouse reports to:
1. Identify JavaScript errors and their root causes
2. Find slow or failing network requests (API calls, assets)
3. Diagnose Core Web Vital issues (LCP, CLS, INP)
4. Detect frontend performance anti-patterns
5. Identify security issues (CORS, CSP, mixed content)
6. Produce a downloadable report with prioritized fixes

---

## Step 1: Read the Input

### Browser console output (pasted text)
Work directly — scan for error/warning/info lines.

### HAR file (`.har`)
JSON format — can be large:
```python
import json
with open('/mnt/user-data/uploads/<file>', 'r', errors='replace') as f:
    har = json.load(f)
entries = har['log']['entries']
print(f"Total requests: {len(entries)}")
# Sort by time taken
slow = sorted(entries, key=lambda e: e['time'], reverse=True)[:20]
for e in slow:
    print(f"{e['time']:.0f}ms | {e['response']['status']} | {e['request']['url'][:100]}")
```

### Network log text export
```bash
head -200 /mnt/user-data/uploads/<file>
grep -i "error\|fail\|404\|500\|503\|timeout\|cors\|blocked" /mnt/user-data/uploads/<file> | head -50
```

### Lighthouse report (JSON or HTML)
```python
import json
report = json.load(open('/mnt/user-data/uploads/<file>'))
# Extract scores
cats = report.get('categories', {})
for k, v in cats.items():
    print(f"{k}: {v['score']*100:.0f}")
# Extract opportunities
audits = report.get('audits', {})
for k, v in audits.items():
    if v.get('score', 1) < 0.9 and v.get('details'):
        print(f"FAIL: {v['title']} — {v.get('displayValue','')}")
```

---

## Step 2: Error Classification

Scan all console lines and classify every error/warning:

### JavaScript Errors
```
Uncaught TypeError: Cannot read properties of undefined (reading 'name')
  at OrderList.jsx:45
```
| Error Type | Common Cause | Fix |
|---|---|---|
| `TypeError: Cannot read properties of undefined` | Async data not yet loaded when component renders | Add null guard / optional chaining (`?.`), loading state |
| `TypeError: X is not a function` | Wrong import, version mismatch, calling non-function | Check import, check API contract |
| `ReferenceError: X is not defined` | Missing import, typo, wrong scope | Add import, check variable scope |
| `SyntaxError` | Parse error — bad JSON, template literal issue | Fix syntax; validate JSON with `JSON.parse` try/catch |
| `Uncaught (in promise)` | Unhandled promise rejection | Add `.catch()` or `try/catch` in async functions |
| `ChunkLoadError` | Lazy-loaded code chunk failed to load | Network issue or deployment mismatch; add error boundary + retry |
| `Maximum call stack size exceeded` | Infinite recursion | Find circular call path |

### Network Errors
| Error | Meaning | Fix |
|---|---|---|
| `Failed to fetch` / `net::ERR_FAILED` | Network unreachable, CORS, or server down | Check network, CORS headers, server status |
| `net::ERR_BLOCKED_BY_CLIENT` | Ad blocker / browser extension blocked | Expected for some analytics; fix if blocking app functionality |
| `CORS error` | Missing `Access-Control-Allow-Origin` header | Add CORS headers on server; check preflight OPTIONS response |
| `net::ERR_CONNECTION_REFUSED` | Server not running or wrong port | Check server, check API base URL config |
| `net::ERR_CERT_*` | TLS certificate issue | Fix certificate, check expiry |
| `Mixed Content` | HTTPS page loading HTTP resource | Serve all resources over HTTPS |
| `CSP violation` | Content Security Policy blocked resource | Update CSP headers to allow legitimate sources |
| 401 Unauthorized | Missing or expired auth token | Check token refresh logic, session management |
| 403 Forbidden | Valid auth but insufficient permissions | Check RBAC, check request headers |
| 404 Not Found | Wrong URL, deployment issue, wrong environment | Check URL construction, API versioning, env config |
| 429 Too Many Requests | Rate limited | Add retry with backoff; reduce request frequency |
| 500/502/503/504 | Server-side error | Check server logs; this is a backend issue |

---

## Step 3: Network Performance Analysis

For each network request, extract:
- URL (especially API endpoints)
- HTTP method
- Status code
- **Duration** (most important)
- Request/response size
- Timing breakdown (DNS, connect, SSL, TTFB, download)

### Timing breakdown analysis:
```
Request to /api/orders
  DNS:      2ms   (normal)
  Connect:  0ms   (keep-alive, good)
  SSL:      0ms   (keep-alive, good)
  Wait:     842ms ← TTFB — server processing time (HIGH)
  Receive:  12ms  (small response, fast)
  Total:    856ms
```

| Phase | Slow Threshold | Diagnosis |
|---|---|---|
| DNS | > 50ms | DNS not cached; use DNS prefetch `<link rel="dns-prefetch">` |
| Connect | > 100ms | No keep-alive; far server; use HTTP/2, CDN |
| SSL/TLS | > 100ms | TLS handshake; use session resumption, HTTP/2 |
| Wait (TTFB) | > 200ms | Server processing slow — check backend, DB queries, caching |
| Receive | > 200ms | Response payload large — compress, paginate, trim fields |

### Waterfall patterns:

**Waterfall blocking** (requests sequential when they could be parallel):
```
Request 1: /api/user        [████████]
Request 2: /api/orders               [████████]   ← starts only after 1 finishes
Request 3: /api/products                      [████████]
```
→ Requests are dependent (result of 1 needed by 2). Fix: batch into single API call, or use `Promise.all()` if not dependent.

**Too many small requests** (N+1 on the frontend):
```
GET /api/product/1   23ms
GET /api/product/2   21ms
GET /api/product/3   25ms
... × 50 more
```
→ Fetching each item individually. Fix: batch API endpoint `GET /api/products?ids=1,2,3,...`

**Large payload**:
```
GET /api/orders  →  Response: 4.2MB, 3,200ms receive
```
→ Fetching too much data. Fix: pagination, field projection (`?fields=id,status,date`), compression.

**Duplicate requests**:
```
GET /api/user  → called 8 times during page load
```
→ Missing request deduplication / caching. Fix: SWR/React Query with dedup, or HTTP caching headers.

**Third-party scripts blocking render**:
```
analytics.js       [████████████████████] 1,200ms
ads-script.js      [████████████████]      900ms
chat-widget.js     [████████████]           700ms
```
→ Third-party scripts on critical path. Fix: `async`/`defer` attributes, load after `DOMContentLoaded`.

---

## Step 4: Core Web Vitals Analysis

If Lighthouse report or CrUX data is provided:

### LCP — Largest Contentful Paint (target: < 2.5s)
The render time of the largest visible element (hero image, heading).

| LCP value | Rating |
|---|---|
| < 2.5s | Good 🟢 |
| 2.5s – 4.0s | Needs Improvement 🟡 |
| > 4.0s | Poor 🔴 |

**Common LCP causes and fixes:**
- Slow server response (TTFB > 800ms) → fix backend, use CDN
- Render-blocking resources → defer non-critical CSS/JS
- Large unoptimized image → compress, use WebP/AVIF, add `fetchpriority="high"` on LCP image
- Image not preloaded → add `<link rel="preload" as="image" href="hero.jpg">`
- Lazy loading on LCP image → remove `loading="lazy"` from the LCP element

### CLS — Cumulative Layout Shift (target: < 0.1)
Measures unexpected visual movement of page content.

**Common CLS causes and fixes:**
- Images without dimensions → always set `width` and `height` attributes
- Ads / embeds without reserved space → reserve space with `min-height`
- Web fonts causing FOUT → use `font-display: optional` or preload fonts
- Dynamic content injected above existing content → insert below, or reserve space
- Animations using `top/left` → use `transform: translate()` instead (doesn't trigger layout)

### INP — Interaction to Next Paint (target: < 200ms)
Responsiveness to clicks/taps/keyboard (replaced FID in 2024).

**Common INP causes and fixes:**
- Long tasks blocking main thread → break with `setTimeout(fn, 0)` or `scheduler.yield()`
- Heavy event handlers → debounce input handlers, move work off main thread
- Large DOM → reduce DOM size (< 1,500 nodes ideal); avoid deep nesting
- Expensive renders on every interaction → memoize components, use virtualization for long lists
- Synchronous localStorage in event handler → use async storage, move to worker

### FCP — First Contentful Paint (target: < 1.8s)
Time to first content rendered.

**Fixes:** eliminate render-blocking resources, inline critical CSS, preconnect to origins.

### TBT — Total Blocking Time (target: < 200ms)
Sum of all long tasks (> 50ms) between FCP and TTI.

**Fixes:** code splitting, lazy loading, remove unused JS, move computation to Web Workers.

---

## Step 5: React / Framework-Specific Issues

If React DevTools Profiler data or console warnings are present:

### React warnings:
```
Warning: Each child in a list should have a unique "key" prop.
```
→ Add stable unique keys to list items. Never use array index as key if list is sorted/filtered.

```
Warning: Cannot update a component (`App`) while rendering a different component (`Child`)
```
→ State update during render. Move `setState` to `useEffect` or event handler.

```
Warning: Can't perform a React state update on an unmounted component.
```
→ Async operation completing after component unmounts. Use cleanup in `useEffect` return function, or `AbortController`.

### React performance patterns:
- **Re-rendering on every parent render**: wrap component in `React.memo()`
- **New object/array/function on every render causing re-renders**: use `useMemo()`, `useCallback()`
- **Large list rendering all items**: use virtualization (`react-virtual`, `react-window`)
- **Heavy computation in render**: move to `useMemo()` with correct deps
- **Context causing all consumers to re-render**: split context by domain, use `useMemo` on value

---

## Step 6: Security Issues

Flag any of these:
- **Mixed content**: HTTP resources on HTTPS page → upgrade to HTTPS
- **CORS misconfiguration**: `Access-Control-Allow-Origin: *` on authenticated endpoints → specify allowed origins
- **CSP missing or too permissive**: `unsafe-inline`, `unsafe-eval` → tighten CSP
- **Sensitive data in URL**: auth tokens, API keys, PII in query strings → move to headers/body
- **Console.log with sensitive data**: passwords, tokens logged → remove before production
- **Exposed API keys in JS**: keys visible in bundle → use environment variables + backend proxy

---

## Step 7: Findings and Recommendations

```
Finding N — [Severity] — [Category: Error / Network / Performance / Security]
Evidence:      <specific console line, URL, timing, metric value>
Root cause:    <what is actually happening>
User impact:   <what the user experiences>
Recommendations:
  1. <Specific code or config change>
  2. <How to verify the fix>
```

**Severity:**
- 🔴 Critical: App broken/unusable, data not loading, security issue
- 🟠 High: Significant slowness (> 2s API calls), Core Web Vitals failing
- 🟡 Medium: Console errors not breaking UX, moderate performance issue
- 🟢 Low: Best practice gap, minor inefficiency

---

## Step 8: Inline Summary

```
## UI Console / Network Analysis

**Source:** <console log / HAR / Lighthouse> | **Page/App:** <if identifiable>

### Error Summary
| Type | Count | Most Critical |
|---|---|---|
| JS Errors | N | TypeError in OrderList.jsx:45 |
| Failed Requests | N | 3× 500 on /api/orders |
| CORS Errors | N | |

### Slowest Requests
| URL | Method | Status | Duration | Issue |
|---|---|---|---|---|

### Core Web Vitals (if available)
| Metric | Value | Target | Status |
|---|---|---|---|

### Findings
1. 🔴 [Critical] ...
```

---

## Step 9: Generate Word Report

Read `/mnt/skills/public/docx/SKILL.md`.

Report sections: Cover, Error Summary, Slow Network Requests (table), Core Web Vitals, Security Issues, Findings with code snippets, Performance Budget Recommendations.

```bash
node /home/claude/ui_console_report.js
cp /home/claude/ui_console_report.docx /mnt/user-data/outputs/ui_console_report.docx
```

---

## Key principles

- **TTFB > 200ms is a backend problem** — don't tune frontend if the server is slow
- **Errors before data loads cause cascade failures** — fix auth/API errors before performance
- **Third-party scripts are often the biggest LCP culprit** — audit ruthlessly
- **N+1 on the frontend is as bad as N+1 on the backend** — batch API calls
- **Core Web Vitals are user experience, not just metrics** — poor CLS/INP means users are actively frustrated
