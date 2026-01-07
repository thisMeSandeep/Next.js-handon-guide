# Next.js App Router – Personal Revision Guide

*(with Better Auth integration)*

> **Audience**: Me (experienced developer)
> **Goal**: Fast recall of Next.js behavior, boundaries, pitfalls, and correct architecture when starting a new project.

---

## Quick Links

- [1. Core Mental Model](#1-core-mental-model-read-this-first)
- [2. Server vs Client Components](#2-server-vs-client-components-execution-rules)
  - [Server Components](#server-components)
  - [Client Components](#client-components)
  - [Composition Rules](#composition-rules)
- [3. Server Actions vs Route Handlers](#3-server-actions-vs-route-handlers-important-distinction)
  - [Server Actions](#server-actions)
  - [Route Handlers](#route-handlers-appapi)
  - [Rule of Thumb](#rule-of-thumb)
- [4. Separation of Concerns (DAL + Services)](#4-separation-of-concerns-dal--services)
  - [Data Access Layer](#data-access-layer-dal)
  - [Service / Action Layer](#service--action-layer)
- [5. Rendering & Caching Strategies](#5-rendering--caching-strategies-this-is-where-bugs-hide)
  - [CSR (Client-side Rendering)](#csr-client-side-rendering)
  - [SSR (Server-side Rendering)](#ssr-server-side-rendering)
  - [SSG (Static Site Generation)](#ssg-static-site-generation)
  - [ISG / ISR](#isg--isr-incremental-static-generationregeneration)
  - [PPR (Partial Page Rendering)](#ppr-partial-page-rendering)
- [6. Auth Boundaries](#6-auth-boundaries-in-nextjs)
- [7. Better Auth with Next.js](#7-better-auth-with-nextjs-separate-section)
  - [Client-side Calls](#client-side-calls)
  - [Server-side Calls](#server-side-calls)
  - [Cookies + Server Actions](#cookies--server-actions-critical)
- [8. Session Management](#8-session-management-better-auth)
  - [Rate Limiting](#rate-limiting)
- [9. Fetching Session Data](#9-fetching-session-data)
- [10. Additional Fields in Session](#10-additional-fields-in-session-object)
- [11. Auto Unauthorized Handling](#11-auto-unauthorized-handling)
- [12. Email Verification Flow](#12-email-verification-flow-better-auth)
- [13. Common Failure Modes](#13-common-failure-modes-debug-checklist)
- [14. Project Startup Checklist](#14-project-startup-checklist)
- [Addendum Intro](#addendum--commonly-missed-but-critical-nextjs--better-auth-topics)
  - [A. Next.js Core Features](#a-nextjs-core-features-you-probably-missed-but-will-need)
    - [Middleware](#1-middleware-request-lifecycle-control)
    - [Edge vs Node Runtime](#2-edge-vs-node-runtime-silent-killer)
    - [Request Context APIs](#3-request-context-apis-scope-rules)
    - [Redirects vs Navigation](#4-redirects-vs-navigation-correct-usage)
    - [Error Boundaries](#5-error-boundaries-app-router-style)
    - [Route Groups & Parallel Routes](#6-route-groups--parallel-routes)
    - [Streaming & Suspense](#7-streaming--suspense-default-not-optional)
    - [Build vs Runtime Thinking](#8-build-vs-runtime-thinking)
  - [B. Better Auth – Important](#b-better-auth--commonly-missed-but-important)
  - [C. Things You Don’t Need](#c-things-you-dont-need-yet)
  - [D. Unknown Unknowns Rule](#d-the-unknown-unknowns-rule)
  - [E. Evolve This Guide](#e-how-to-evolve-this-guide-important)

---

## 1. Core Mental Model (Read This First)

Next.js App Router is **not a frontend framework with a backend bolted on**.
It is a **React-first full-stack runtime** with strict execution boundaries.

Key principles:

* **Execution matters more than location**
* **Server code is never shipped to the client**
* **Caching is the default**
* **Mutations are special**
* **Auth is request-scoped**

If something breaks, it’s usually because **a boundary was crossed unintentionally**.

---

## 2. Server vs Client Components (Execution Rules)

### Server Components

* Default in the `app/` directory
* Run **only on the server** (never shipped to the browser)
* Purpose: read secure things (DB, headers, cookies) and assemble the initial UI tree
* Can:

  * Access DB
  * Read cookies / headers
  * Call internal services
* Cannot:

  * Use browser APIs
  * Use hooks like `useState`, `useEffect`

```ts
// Server Component
const session = await getSession()
```

### Client Components

* Marked with `"use client"`
* Run in the browser
* Purpose: handle interactive UX (state, effects, event handlers)
* Can:

  * Handle UI state
  * Use effects
* Cannot:

  * Access secrets
  * Access DB directly

### Composition Rules

* ✅ Server → Client (allowed)
* ❌ Client → Server (not allowed)

**Components vs Rendering strategies**: Server/Client components are about *where code executes* (server process vs browser). Rendering strategies (CSR/SSR/SSG/ISG/PPR) are about *when and how* HTML/data are produced and cached. You can mix: a page rendered via SSR can include both server and client components.

---

## 3. Server Actions vs Route Handlers (Important Distinction)

### Key Truth

> **Server Actions are optional. Route Handlers are sufficient.**

Server Actions are **not more powerful** — they are **more ergonomic** for React apps.

---

### Server Actions

Best for:

* Forms
* Internal UI mutations
* React-only apps
* Tight UI ↔ mutation coupling

```ts
"use server"
export async function createPost(data) {
  await db.post.create({ data })
}
```

Called directly from:

* Server Components
* Client Components (via React wiring)

---

### Route Handlers (`app/api/*`)

Best for:

* Public APIs
* Mobile apps
* Webhooks
* External consumers
* Clear HTTP contracts

```ts
export async function POST(req: Request) {
  const body = await req.json()
}
```

---

### Rule of Thumb

| Use case             | Prefer        |
| -------------------- | ------------- |
| Internal UI mutation | Server Action |
| External consumer    | Route Handler |
| Forms                | Server Action |
| SDK / mobile         | Route Handler |

---

## 4. Separation of Concerns (DAL + Services)

### Data Access Layer (DAL)

* Talks **only** to the database
* No auth
* No request context

```ts
// dal/user.ts
export function getUserById(id: string) {
  return db.user.findUnique({ where: { id } })
}
```

---

### Service / Action Layer

* Enforces:

  * Auth
  * Validation
  * Business rules
* Calls DAL

```ts
"use server"
export async function updateProfile(data) {
  requireAuth()
  return updateUser(data)
}
```

**Never put DB calls directly in UI components.**

---

## 5. Rendering & Caching Strategies (This Is Where Bugs Hide)

### CSR (Client-side Rendering)

* HTML is a minimal shell; data fetched and rendered in the browser after load.
* Great for highly interactive, client-heavy flows where SEO is less critical.
* Example: an internal dashboard page with heavy client-side state and charts.

---

### SSR (Server-side Rendering)

```ts
export const dynamic = "force-dynamic"
```

* HTML is generated **per request** on the server with fresh data.
* Best for auth-aware pages and personalized content; ensures request-scoped auth.
* Example: a user profile page that must reflect the latest session and permissions.

---

### SSG (Static Site Generation)

```ts
export const revalidate = false // or omit for fully static at build time
```

* HTML generated at build time; zero per-request work; fastest, cache-friendly.
* Best for content that rarely changes and is the same for all users.
* Example: a marketing homepage with copy and images that change only on deploy.

---

### ISG / ISR (Incremental Static Generation/Regeneration)

```ts
export const revalidate = 60 // seconds
```

* Starts with static output; Next.js revalidates in the background on interval.
* Good balance for mostly-static content that needs periodic freshness.
* Example: a blog index that updates every few minutes as new posts are published.

---

### PPR (Partial Page Rendering)

* Render a static shell while streaming dynamic islands inside it.
* Combine cacheable sections with request-specific sections for faster TTFB.
* Example: a product page with a static description but live inventory/pricing streamed.

---

### React `cache()` helper (memoizing server functions)

Fine-grained server memoization for repeated calls in a request or across requests:

```ts
import { cache } from "react"

export const getUser = cache(async (id) => {
  return db.user.findUnique({ where: { id } })
})
```

* Cache functions, not components; useful with PPR/SSR to avoid duplicate fetches.

## 6. Auth Boundaries in Next.js

Auth logic **must run inside a request scope**.

Valid places:

* Server Components
* Server Actions
* Route Handlers
* Middleware

Invalid:

* Build time
* Static rendering without `force-dynamic`

---

## 7. Better Auth with Next.js (Separate Section)

### Client vs Server Usage

#### Client-side calls

```ts
authClient.signIn()
```

Use when:

* You need immediate UI feedback
* Optimistic UX
* Form-heavy flows

---

#### Server-side calls

```ts
auth.api.getSession({ headers: await headers() })
```

Use when:

* Secrets must stay server-only
* DB/cache work is involved
* You want thinner client bundles

---

### Cookies + Server Actions (CRITICAL)

If a Server Action:

* Signs in
* Signs up
* Refreshes session

👉 **You MUST enable the `nextCookies` plugin**

Otherwise:

* Cookie writes silently fail
* Sessions appear broken

---

## 8. Session Management (Better Auth)

* Sessions stored via **cookies + JWT**
* Default refresh: **7 days**
* Fully configurable via Better Auth config

### Rate Limiting

* Auth endpoints are auto rate-limited
* Protects against brute-force attacks
* Can be customized if needed

---

## 9. Fetching Session Data

### On the Server

```ts
auth.api.getSession({ headers: await headers() })
```

### On the Client

```ts
authClient.getSession()
```

**Rule**:

* Server = authoritative
* Client = convenience

---

## 10. Additional Fields in Session Object

Use:

```ts
inferAdditionalFields
```

Allows:

* Custom user data in session
* Typed access on client

Important:

* Only add **frequently needed** fields
* Don’t overload the session

---

## 11. Auto Unauthorized Handling

Next.js provides:

```ts
unauthorized()
```

Behavior:

* Throws a special error
* Redirects to `unauthorized.tsx`

Create:

```
app/unauthorized.tsx
```

Use this instead of manual redirects.

---

## 12. Email Verification Flow (Better Auth)

* Built-in email verification support
* Requires implementing:

  * `sendVerificationEmail`
* Customize:

  * Email content
  * Callback URL

Typical flow:

1. User signs up
2. Verification email sent
3. User clicks link
4. Redirect to `email-verified.tsx`

---

## 13. Common Failure Modes (Debug Checklist)

When something breaks, ask:

1. Is this running in **static vs dynamic** mode?
2. Am I inside a **request scope**?
3. Is a **cookie stale or corrupted**?
4. Did I change auth secrets without clearing cookies?
5. Am I accidentally in **edge runtime** with Prisma?

Most “Next.js bugs” are boundary violations.

---

## 14. Project Startup Checklist

Before writing features:

* [ ] Decide Server Actions vs Routes
* [ ] Define DAL
* [ ] Lock auth secrets
* [ ] Decide caching strategy
* [ ] Mark auth pages `force-dynamic`
* [ ] Clear cookies after auth config changes


-----------------------

# Additonal features

# Addendum — Commonly Missed but Critical Next.js & Better Auth Topics

This section exists specifically to cover **things you don’t use every day but WILL forget**, and which usually break apps when forgotten.

---

## A. Next.js Core Features You Probably Missed (But Will Need)

### 1. Middleware (Request Lifecycle Control)

**Why it matters**
Middleware runs **before everything**:

* Before Server Components
* Before Route Handlers
* Before Server Actions

Use it for:

* Auth gating
* Redirects
* Locale
* Feature flags

```ts
// middleware.ts
export function middleware(req: NextRequest) {
  if (!req.cookies.get("session")) {
    return NextResponse.redirect(new URL("/login", req.url))
  }
}
```

⚠️ Runs in **Edge Runtime** by default
⚠️ No DB access
⚠️ No Prisma

---

### 2. Edge vs Node Runtime (Silent Killer)

By default:

* Middleware → Edge
* Route Handlers → Edge (sometimes)
* Server Components → Node

If you use Prisma:

```ts
export const runtime = "nodejs"
```

Failure symptom:

> Random crashes inside node_modules

---

### 3. Request Context APIs (Scope Rules)

These only work **inside a request**:

* `headers()`
* `cookies()`
* `draftMode()`

They **fail silently** in:

* Static rendering
* Build time
* Cached functions

This explains 50% of auth bugs.

---

### 4. Redirects vs Navigation (Correct Usage)

Server:

```ts
redirect("/login")
```

Client:

```ts
router.push("/login")
```

Using the wrong one:

* breaks streaming
* breaks cache
* causes hydration issues

---

### 5. Error Boundaries (App Router Style)

You likely missed:

* `error.tsx`
* `not-found.tsx`
* `loading.tsx`

These are **route-scoped**, not global.

This is how you avoid try/catch hell.

---

### 6. Route Groups & Parallel Routes

These are **structural tools**, not routing features.

Use them to:

* Share layouts
* Separate auth vs app UI
* Build dashboards

Example:

```
(app)
(auth)
```

Parallel routes:

* Modals
* Side panels
* Tabs

---

### 7. Streaming & Suspense (Default, Not Optional)

Everything is streamed by default.

You should:

* Use `Suspense` intentionally
* Place boundaries around slow data

```tsx
<Suspense fallback={<Skeleton />}>
  <UserProfile />
</Suspense>
```

---

### 8. Build vs Runtime Thinking

Ask constantly:

> “Is this evaluated at build time or request time?”

Most Next.js confusion comes from **forgetting this distinction**.

---

## B. Better Auth – Commonly Missed but Important

### 1. Cookie Corruption & Secret Rotation

If you change:

* Auth secret
* Cookie config
* Adapter

👉 **Clear cookies immediately**

Otherwise:

* Base64 decode errors
* Session failures inside node_modules

(You already hit this — good lesson.)

---

### 2. Cookie Path, Domain & SameSite

Misconfigured cookies cause:

* Login works → refresh breaks
* Works in dev → fails in prod

Always verify:

* `sameSite`
* `secure`
* `domain`

---

### 3. Auth + Middleware Coordination

Middleware:

* Cannot read DB
* Can read cookies

Better Auth:

* Session validation happens later

Rule:

> Middleware only *gatekeeps*, it does NOT validate deeply.

---

### 4. Session Freshness vs Caching

Never cache:

* Authenticated pages
* User-specific data

Always combine:

```ts
export const dynamic = "force-dynamic"
```

with:

```ts
getSession()
```

---

### 5. Logout Is a Write Operation

Logout:

* Mutates cookies
* Requires Server Action or Route Handler

Client-only logout often **appears to work but doesn’t clear server state**.

---

### 6. Multiple Tabs & Session Sync

Better Auth uses cookies → shared across tabs.

But:

* Client state must re-fetch session
* Don’t trust in-memory auth state

---

## C. Things You Don’t Need (Yet)

These are often overused:

* `generateStaticParams` (unless content-heavy)
* `rewrites` (unless proxying)
* Custom Webpack config
* Manual cache invalidation everywhere

---

## D. The “Unknown Unknowns” Rule

If something:

* breaks randomly
* works after refresh
* crashes in node_modules

It’s almost always:

* caching
* runtime mismatch
* request scope violation
* stale cookies

Not the library.

---

## E. How to Evolve This Guide (Important)

Do NOT aim for completeness.

Instead:

* Every time you debug a weird issue
* Add **one bullet** under the right section




