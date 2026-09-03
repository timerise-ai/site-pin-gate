---
name: site-pin-gate
description: >
  Put a shared-PIN gate in front of an entire Next.js site from the
  proxy/middleware layer: one env var arms it, a self-contained unlock page
  sets an HMAC-derived HttpOnly cookie, and the app behind it stays untouched.
  Use when: (1) a pre-launch, staging, client-preview or investor-preview site
  must be hidden from the public and from crawlers without adding user
  accounts, (2) the user mentions: site PIN, password-protect the site, access
  code, pre-launch gate, coming-soon lock, staging password, SITE_PIN, "hide
  the site behind a PIN", "lock the preview deployment", "temporary password
  page", (3) an existing PIN or password gate in middleware.ts or proxy.ts
  needs auditing. Carries the fixes for defects found live in the source (an
  open redirect after unlock, a cookie that leaks the PIN, a 307 that re-POSTs
  the PIN, a 500 on non-form bodies) plus an attempt budget and tests. Next.js
  App Router, proxy.ts or middleware.ts; not user authentication and not
  per-route authorization.
---

# Site PIN gate — one PIN in front of the whole site

A PIN gate is the smallest possible access control: one shared secret, one
cookie, zero user accounts. Its whole value is that the app behind it never
knows it exists. The insight that shapes this module is that **everything
dangerous lives in the two places the gate touches the outside world** — the
return path it redirects to, and the cookie it hands out — and the source
module got both wrong while looking perfectly reasonable.

## When to use

A public-by-default Next.js site that must be hidden for a while: before
launch, during a client or investor review, on a staging domain. Everyone who
should see it can be told one PIN. The gate sits in `proxy.ts` (Next 16) or
`middleware.ts` (Next 13–15) and is removed by unsetting one env var.

## When NOT to use

- **Real users with real accounts.** Use the host's auth (Clerk, NextAuth,
  Supabase Auth). A shared PIN cannot be revoked for one person.
- **Per-route or per-tenant authorization.** This gate is all-or-nothing by
  path matcher; it has no notion of who is asking.
- **Only preview deployments, on Vercel, for team members.** Vercel
  Authentication does that with no code. See the comparison in
  [operations.md](references/operations.md).
- **Protecting an API consumed by machines.** Give them a key; a cookie form
  is the wrong shape.

## Architecture

```
request ─► proxy.ts / middleware.ts (matcher decides what is even seen)
              │
              ├─ SITE_PIN unset ──────────────────────────────► app (gate off)
              │
              ├─ cookie == HMAC(secret, pin) ──────────────────► app
              │
              ├─ GET  /__unlock ─────────────► 303 /
              ├─ POST /__unlock  bad body ───► 400 gate page
              │                  over budget ► 429 gate page + Retry-After
              │                  wrong PIN ──► 401 gate page (return path kept)
              │                  right PIN ──► 303 safe return path + Set-Cookie
              │
              └─ anything else ───────────────► 401 gate page, no-store, noindex
```

## Critical facts

1. **An unset PIN is an open site, silently.** The most likely production
   failure is the env var set in one environment and not another. Smoke-test
   every environment; the check is one `curl`.
2. **The matcher is the security boundary.** Whatever it excludes is public.
   Build assets always are; sitemap, robots, OG images and `/api` are public
   only if you exclude them — the source did, and shipped its route list.
3. **The cookie is a bearer token derived from the PIN.** Rotating the PIN
   logs every browser out with no store. Without `SITE_GATE_SECRET` it is a
   plain fingerprint of the PIN, and a leaked cookie is a leaked PIN.
4. **The PIN's length is the real defence.** The attempt budget is per server
   instance and best-effort. A four-digit PIN is ten thousand requests.
5. **The gate page is self-contained by design.** The proxy runs before the
   app, with no stylesheet, no components, no i18n provider. Inline CSS and a
   strings table are not laziness.

## Hard rules

> **Never validate a return path with `startsWith('/')`.** `//evil.example`,
> `/\evil.example` and `///evil.example` all pass it and all leave the site.
> Resolve against the origin and compare origins.

> **Never answer a successful form POST with a 307.** The browser re-POSTs the
> body — PIN included — to the page it lands on, and on every redirect after.
> Use 303.

> **Never compare the PIN or the cookie with `===`.** Route both through a
> constant-time comparison of equal-length digests.

> **Never put the PIN, or an unkeyed hash of it, in the cookie.** A cookie the
> operator can read is a cookie an operator's laptop can leak.

> **Never let `request.formData()` throw to the platform.** A JSON body or a
> bare POST is a 500 page with a stack trace, not a rejected attempt.

> **Never mount the unlock path under a locale prefix or an app route.** The
> gate handles it before routing; a colliding page would never render.

## Quick start

0. Probe the host and fill in the seam table —
   [adaptation.md](references/adaptation.md).
1. Copy the config, pure helpers and attempt store —
   [module.md](references/module.md).
2. Copy the page renderer and the handler; wire them into `proxy.ts` or
   `middleware.ts` and choose the matcher — [handler.md](references/handler.md).
3. Set `SITE_PIN` and `SITE_GATE_SECRET` per environment; run the smoke checks
   — [operations.md](references/operations.md).
4. Run the shipped tests in the host's runner — [testing.md](references/testing.md).

## Reference directory

| Scenario | Trigger keywords | Reference |
|---|---|---|
| Fitting this into an existing app | adapt, seam, proxy.ts, middleware.ts, matcher, next-intl, cookie name, unlock path, strings, Redis | [adaptation.md](references/adaptation.md) |
| Config, token, return path, budget | SITE_PIN, SITE_GATE_SECRET, HMAC, constantTimeEqual, safeReturnPath, open redirect, rate limit, x-forwarded-for | [module.md](references/module.md) |
| The page, the handler, the wiring | renderGatePage, createSiteGate, 303, 401, 429, formData, Set-Cookie, x-robots-tag, inline CSS | [handler.md](references/handler.md) |
| Running it in production | env per environment, Vercel, rotate PIN, kill switch, smoke test, sitemap, robots, OG image, /api, Deployment Protection, lock endpoint, multiple PINs | [operations.md](references/operations.md) |
| Proving it | vitest, bun test, NextRequest, test cases | [testing.md](references/testing.md) |
| Why the templates differ from the source | provenance, defect, audit, kept deliberately, porting | [provenance.md](references/provenance.md) |
