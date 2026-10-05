# Adaptation contract

Everywhere this module touches its host. Fill in the right-hand column from
the target app **before** generating code. There is no domain vocabulary to
rename, no tenant scope, no auth guard to call and no data access: the gate
*is* the guard, and it keeps no state beyond one cookie and one in-memory
attempt counter.

| Seam | This skill ships | Your app supplies | Where it plugs in |
|---|---|---|---|
| Secrets | `SITE_PIN`, `SITE_GATE_SECRET`, `SITE_GATE_BRAND` env vars | Its env management, per environment | `readSiteGateConfig` in `config.ts` |
| Entry point | `createSiteGate()` + a `proxy.ts` template | `proxy.ts` (Next 16) or `middleware.ts` (Next 13 to 15), and whatever it already runs there | [handler.md](handler.md) |
| Matcher | "everything but build assets" | Its own exclusions: webhooks, health checks, an already-public API | `config.matcher`; a bypass list in the proxy when the matcher already serves other jobs |
| Cookie name, unlock path | `site_access`, `/__unlock` | Names that collide with nothing; the unlock path outside any locale prefix | `SITE_GATE_DEFAULTS`, or `readSiteGateConfig(env, overrides)` |
| Strings | `GateStrings` type, English table | One table per locale it serves | `createSiteGate({ strings })` |
| Locale detection | Locale cookie, then `Accept-Language`, then default | Its locale cookie name and default locale, or its own negotiation function | `createSiteGate({ localeCookieName, defaultLocale })` or `createSiteGate({ resolveLocale })` |
| Text direction | Left-to-right | Which of its locales are right-to-left | `createSiteGate({ localeDir })` |
| Styling | Inline CSS on custom properties, `data-site-gate` hooks | Its palette in the `:root` block; nothing else | `renderGatePage` in `page.ts` |
| Attempt store | In-memory, per instance | A shared store (Redis, a table) for a hard global cap | `createSiteGate({ attempts })` |
| Logging | `console.warn` | Its logger, if it has one | `createSiteGate({ log })` |
| Tests | vitest-style files, also run under `bun test` | Its runner | [testing.md](testing.md) |

## Host probe

Run before writing anything:

```bash
grep -E '"(next|next-intl|vitest)"' package.json
ls src/proxy.ts src/middleware.ts proxy.ts middleware.ts 2>/dev/null   # which convention, and is one already there
grep -n "matcher" src/proxy.ts src/middleware.ts proxy.ts middleware.ts 2>/dev/null
grep -rn "NEXT_LOCALE\|localeCookie" src/i18n src/proxy.ts 2>/dev/null | head -3   # locale cookie name
grep -rn "rate-limit\|rateLimit\|ratelimit" src/lib 2>/dev/null | head -3         # an attempt store to reuse?
grep -rln "negotiate\|acceptLanguage\|accept-language" src/lib src/i18n lib i18n 2>/dev/null | head -3   # locale negotiation to reuse?
# Machine callers: everything here must stay reachable without the cookie.
ls -d app/api/*/ src/app/api/*/ 2>/dev/null                                       # webhooks, cron, health, a public API
grep -n "crons\|path:" vercel.ts vercel.json 2>/dev/null | head                   # scheduled requests
find app src/app -name route.ts -not -path "*/api/*" 2>/dev/null                  # feeds and callbacks outside /api
cat CLAUDE.md AGENTS.md 2>/dev/null | head -60
```

Write the machine callers down before choosing the matcher: provider
webhooks (payments, KYC, email events, tracking, video, helpdesk, calendar
push), cron paths and any manual trigger URL for them, health probes, and
feeds a third party polls, such as an ICS calendar feed. None of them holds
the cookie. A browser redirect does: an auth callback, a payment return or a
link in an email reaches the gate in the visitor's own browser, and after the
PIN the unlock returns to the same path with its query, so those stay gated.

Then read the existing proxy or middleware end to end. The gate goes **first**
in it, before locale routing, before auth, before rewrites, because every
one of those may redirect, and a redirect from a locked site leaks the route
it redirects to.

**Add no dependency.** The templates import only `next/server` and Web
Crypto, both already present in every Next.js app.

## Next 16 `proxy.ts` versus Next 13 to 15 `middleware.ts`

The body is identical; only the file name and the export change.

| | Next 16 | Next 13 to 15 |
|---|---|---|
| File | `proxy.ts` (or `src/proxy.ts`) | `middleware.ts` (or `src/middleware.ts`) |
| Export | `export async function proxy(request)` | `export async function middleware(request)` |
| Runtime | Node.js by default | Edge by default; Node.js opt-in from 15.5 |

Web Crypto (`crypto.subtle`) is available on both runtimes, which is why the
token derivation does not import `node:crypto`. Keep it that way if the host is
on the Edge runtime.

## Composing with what is already there

The handler returns a `NextResponse` to send instead of the app, or `null` to
continue. Compose by early return:

```ts
// file: proxy-with-i18n.ts
import type { NextRequest } from 'next/server';
import createIntlMiddleware from 'next-intl/middleware';

import { createSiteGate } from '@/lib/site-gate/handler';
import { routing } from '@/i18n/routing';

const handleI18nRouting = createIntlMiddleware(routing);
const siteGate = createSiteGate({
  localeCookieName: 'NEXT_LOCALE', // next-intl's default
  defaultLocale: routing.defaultLocale,
});

export async function proxy(request: NextRequest) {
  const gate = await siteGate(request);
  if (gate) return gate;
  return handleI18nRouting(request);
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon\\.ico).*)'],
};
```

`next-intl` ships one strings table per locale in the host already; the gate
cannot read it (the proxy has no message provider), so pass the gate strings
explicitly: see "Strings" below.

## Choosing the matcher

The matcher decides what the gate ever sees. Two defensible shapes:

| Matcher | Gated | Public | Use when |
|---|---|---|---|
| `/((?!_next/static\|_next/image\|favicon\\.ico).*)` | Pages, RSC payloads, `/api`, `sitemap.xml`, `robots.txt`, OG images, `public/` files | Build assets only | **Default.** Nothing about the site should be visible |
| `/((?!api\|_next\|_vercel\|.*\\..*).*)` | Pages and RSC payloads | `/api`, every file with an extension, generated images | Third parties must reach `/api` (webhooks) or a public file, and you accept that the sitemap lists every route |

Copy the chosen matcher verbatim. Adding an exclusion (`_next/image`,
`favicon.ico`, `api/...`) makes a path public and needs a reason the task
names; dropping one, as in `/((?!_next/static).*)`, is an unreviewed third
shape that buys nothing. `/:path*` is not a third shape either. Gating `_next/static` keeps nothing from a
crawler, which never holds the cookie and sees only the `401`, and it sends
every build asset through the proxy. "Nothing indexed" is the default matcher;
see *Indexing* in [operations.md](operations.md).

With the default matcher, the site's own `fetch('/api/...')` calls still work:
the browser sends the gate cookie on same-origin requests. What breaks is
anything **without** the cookie: a webhook, an uptime probe, a mobile app.
Exclude those paths explicitly rather than opening all of `/api`:

```ts
// file: matcher-with-allowlist.ts
export const config = {
  matcher: [
    '/((?!_next/static|_next/image|favicon\\.ico|api/webhooks/|api/health).*)',
  ],
};
```

## When the proxy's matcher is shared

A host that already has a proxy usually runs it on nearly every path, for a
session refresh, a CSP nonce or a request id, and its webhooks need that
proxy as much as its pages do. Narrowing that matcher to suit the gate breaks
the other jobs. Keep the host's matcher, and skip the gate in code for the
machine-caller paths from the probe:

```ts
// file: proxy-with-bypass.ts
import { type NextRequest, NextResponse } from 'next/server';

import { createSiteGate } from '@/lib/site-gate/handler';

const siteGate = createSiteGate();

// Callers that cannot type a PIN. Prefixes end in `/`, so `/apix` stays gated.
const PUBLIC_PREFIXES = ['/api/', '/calendar/feed/'] as const;
const PUBLIC_PATHS = new Set(['/api', '/healthz']);

export function isGatePublicPath(pathname: string): boolean {
  return (
    PUBLIC_PATHS.has(pathname) ||
    PUBLIC_PREFIXES.some(prefix => pathname.startsWith(prefix))
  );
}

export async function proxy(request: NextRequest) {
  if (!isGatePublicPath(request.nextUrl.pathname)) {
    const gate = await siteGate(request);
    if (gate) return gate;
  }
  // ...the host's own proxy logic, unchanged
  return NextResponse.next();
}
```

The bypass list is then the security boundary instead of the matcher, with
the same rule: every entry is public, and each needs a reason. Opening all of
`/api` is defensible only when every route under it enforces its own auth;
otherwise list the webhook and cron prefixes one by one. Test the list with
its edges: a bypassed path, its bare prefix, and a sibling such as `/apix`
that must stay gated.

`_next/static` is always public. Client-component code and any string in it
can be read by whoever knows a chunk URL; the chunk URLs are only referenced
from gated HTML, but they are not secret. Do not put secrets in client code
and expect the gate to cover them.

## Strings

The gate page speaks whatever languages you give it. One table per locale,
keyed by the locale tag the host uses:

```ts
// file: lib/site-gate/strings.ts
import { GATE_STRINGS_EN, type GateStrings } from './page';

export const GATE_STRINGS: Record<string, GateStrings> = {
  en: GATE_STRINGS_EN,
  pl: {
    title: 'Dostęp ograniczony',
    prompt: 'Wpisz PIN dostępu, aby kontynuować.',
    label: 'PIN dostępu',
    button: 'Odblokuj',
    errors: {
      wrong: 'Nieprawidłowy PIN. Spróbuj ponownie.',
      rateLimited: 'Zbyt wiele prób. Odczekaj kilka minut i spróbuj ponownie.',
      invalid: 'Nie udało się odczytać żądania. Spróbuj ponownie.',
    },
  },
};
```

Pass it as `createSiteGate({ strings: GATE_STRINGS })`.

### A host that negotiates locales itself

`pickLocale` matches a tag or its base language. That misses script subtags:
a browser sending `zh-CN` has base `zh`, which is not `zh-Hans`. A host with
its own negotiation (profile language, aliases, an enabled-locale list) should
hand it to the gate instead, and say which locales are right-to-left:

```ts
// file: lib/site-gate/instance-with-host-locale.ts
import { negotiateLocale, localeDir } from '@/lib/i18n/negotiate';

import { createSiteGate } from './handler';
import { GATE_STRINGS } from './strings';

export const siteGate = createSiteGate({
  strings: GATE_STRINGS,
  defaultLocale: 'en',
  resolveLocale: request =>
    negotiateLocale({
      cookieLocale: request.cookies.get('NEXT_LOCALE')?.value,
      acceptLanguage: request.headers.get('accept-language'),
    }),
  localeDir, // 'rtl' for ar, he, fa, ur
});
```

The resolver runs in the proxy, before any session exists, so it can read
the cookie and headers only; a profile language stored in the database is out
of its reach. A result the strings table lacks renders the default locale. The page's `lang`
attribute follows the pick. A locale the table lacks falls back to
`defaultLocale`; a locale cookie value the table lacks is ignored.

## Styling

The page is a single HTML document with its CSS inline. This is deliberate:
the proxy has no access to the app's stylesheet or component tree, and the
gate must render when the app behind it is broken. Restyle by editing the
custom properties in the `:root` block (background, foreground, muted, card,
border, accent, error), in both the light and the `prefers-color-scheme: dark`
sets. If the host insists on its own stylesheet, the matcher's default
excludes nothing under `public/`, so a `<link>` to `/gate.css` would itself be
gated; add `gate\\.css` to the matcher exclusion before linking it.

## Cookie name and unlock path

Both are overridable per host. Rename when the defaults collide, or when two
gated apps share a parent domain (cookies do not carry a port, and a cookie
named `site_access` set by one app on `localhost` is presented to the other).
The unlock path must not resolve to an app route or sit under a locale prefix;
`/__unlock` is chosen so neither happens by accident.

```ts
// file: lib/site-gate/instance.ts
import { readSiteGateConfig } from './config';
import { createSiteGate } from './handler';

export const siteGate = createSiteGate({
  config: readSiteGateConfig(process.env, {
    cookieName: 'preview_access',
    unlockPath: '/__preview-unlock',
    cookieMaxAgeSeconds: 60 * 60 * 24 * 7, // one week
    brand: 'Acme Preview',
  }),
});
```

## A shared attempt store

The in-memory store is per server instance. On a platform that fans requests
across many instances an attacker gets `limit` guesses per instance per
window, which still turns unbounded guessing into a crawl but is not a hard
cap. For a hard cap, implement `AttemptStore` over something shared:

```ts
// file: lib/site-gate/redis-attempts.ts
import type { AttemptStore } from './attempts';

type MinimalRedis = {
  incr(key: string): Promise<number>;
  pexpire(key: string, ms: number): Promise<unknown>;
  del(key: string): Promise<unknown>;
};

/** Fixed-window counter: one INCR per attempt, expiry set on the first. */
export function createRedisAttemptStore(
  redis: MinimalRedis,
  limit: number,
  windowMs: number,
  prefix = 'site-gate:attempts:'
): AttemptStore {
  return {
    async hit(key) {
      const n = await redis.incr(prefix + key);
      if (n === 1) await redis.pexpire(prefix + key, windowMs);
      return n <= limit;
    },
    async reset(key) {
      await redis.del(prefix + key);
    },
  };
}
```

This adapter is a design and has never run in production; the in-memory store
is what ships. It compiles against the two-method shape above, which `ioredis`
and the Upstash client both satisfy.

## Order of work

1. Run the host probe above and fill in the seam table.
2. Copy `config.ts`, `core.ts` and `attempts.ts` from [module.md](module.md);
   they have no framework import and nothing to adapt but the defaults.
3. Copy `page.ts` and `handler.ts` from [handler.md](handler.md), restyle the
   `:root` block, and add a strings table per locale.
4. Wire the gate first in `proxy.ts` or `middleware.ts` and choose the
   matcher, or keep a shared one and add the bypass list.
5. Set the env vars per environment and run the smoke checks in
   [operations.md](operations.md).
6. Copy the two suites from [testing.md](testing.md) into the host's runner.

## The non-negotiables

Restated from `SKILL.md`, because this is the file open while the module is
being fitted to a host. None of the six is a style preference, and each is
covered by the suite in [testing.md](testing.md):

1. The return path is resolved against the request origin, never checked with
   `startsWith('/')`.
2. The unlock redirect is a `303`, so the browser never repeats the POST.
3. The cookie is an HMAC keyed by `SITE_GATE_SECRET`, never the PIN and never
   an unkeyed hash of it.
4. Both comparisons are constant-time over equal-length digests.
5. A body that is not a form is answered with `400`, never thrown to the
   platform.
6. The matcher, or the bypass list beside a shared one, is chosen on purpose,
   and the unlock path sits outside every locale prefix and app route.

Everything else is the host app's: cookie name, unlock path, strings,
palette, locale detection, attempt store.

## Adaptation checklist

- [ ] `package.json` read; no dependency added
- [ ] Existing proxy/middleware read; the gate runs first in it
- [ ] Machine callers listed: webhooks, crons and their trigger URLs, probes, polled feeds
- [ ] Matcher chosen deliberately, or the host's kept and a bypass list added;
      webhook and health paths excluded by name, and the list tested at its edges
- [ ] Cookie name and unlock path checked for collisions
- [ ] A strings table for every locale the host serves
- [ ] Locale cookie name and default locale passed in, or the host's resolver
- [ ] `localeDir` passed when any served locale is right-to-left
- [ ] `:root` palette adjusted to the host's brand, both colour schemes
- [ ] `SITE_PIN` and `SITE_GATE_SECRET` set in every environment that should be
      locked, see [operations.md](operations.md)
