# The module: config, pure helpers, attempt store

Three files with no request object and no framework import in two of them.
Everything the handler decides — is this cookie valid, is this return path
safe, has this client used its budget — is a call into here, and every one of
those calls is unit-tested in [testing.md](testing.md).

## Configuration

| Env var | Required | Effect |
|---|---|---|
| `SITE_PIN` | to arm the gate | The PIN. Unset or whitespace disables the gate entirely. Any string; length is the defence, so prefer eight or more characters |
| `SITE_GATE_SECRET` | strongly recommended | Keys the cookie token. Without it the token is a fingerprint of the PIN alone. Any long random string; rotating it logs everyone out |
| `SITE_GATE_BRAND` | no | Heading and `<title>` prefix on the gate page. Defaults to "Restricted" |
| `NODE_ENV` | — | `production` sets the cookie's `Secure` flag |

```ts
// file: lib/site-gate/config.ts
export type SiteGateConfig = {
  /** The shared PIN. Undefined or blank disables the gate entirely. */
  pin: string | undefined;
  /**
   * Server-side secret that keys the cookie token. Without it the cookie is a
   * bare fingerprint of the PIN and a leaked cookie yields the PIN offline.
   */
  secret: string | undefined;
  /** Cookie that marks an unlocked browser. */
  cookieName: string;
  /** Path the gate form posts to. Must not collide with an app route. */
  unlockPath: string;
  /** How long an unlocked browser stays unlocked. */
  cookieMaxAgeSeconds: number;
  /** Failed-attempt budget per client key. */
  attempts: { limit: number; windowMs: number };
  /** Send the cookie with `Secure`. Only turn off for plain-http local dev. */
  secureCookie: boolean;
  /** Shown as the heading of the gate page. */
  brand: string;
};

export const SITE_GATE_DEFAULTS = {
  cookieName: 'site_access',
  unlockPath: '/__unlock',
  cookieMaxAgeSeconds: 60 * 60 * 24 * 30, // 30 days
  attempts: { limit: 5, windowMs: 15 * 60 * 1000 }, // 5 tries per 15 min
} as const;

/**
 * Reads the gate configuration from the environment. Whitespace-only values
 * count as unset: a PIN of " " would otherwise arm a gate nobody can open,
 * because submitted PINs are trimmed and the env value was not.
 */
export function readSiteGateConfig(
  env: Record<string, string | undefined> = process.env,
  overrides: Partial<Omit<SiteGateConfig, 'pin' | 'secret'>> = {}
): SiteGateConfig {
  const pin = env.SITE_PIN?.trim() || undefined;
  const secret = env.SITE_GATE_SECRET?.trim() || undefined;
  return {
    pin,
    secret,
    cookieName: SITE_GATE_DEFAULTS.cookieName,
    unlockPath: SITE_GATE_DEFAULTS.unlockPath,
    cookieMaxAgeSeconds: SITE_GATE_DEFAULTS.cookieMaxAgeSeconds,
    attempts: { ...SITE_GATE_DEFAULTS.attempts },
    secureCookie: env.NODE_ENV === 'production',
    brand: env.SITE_GATE_BRAND?.trim() || 'Restricted',
    ...overrides,
  };
}
```

`readSiteGateConfig` takes the env as a parameter so tests can pass a plain
object; the handler reads `process.env` once at instance start. Next.js does
not inline server-side env vars at build time, so a value changed in the
platform takes effect on the next deployment, not the next request.

## Pure helpers

```ts
// file: lib/site-gate/core.ts
// Pure helpers: no request object, no framework import, fully unit-testable.

const encoder = new TextEncoder();

function toHex(bytes: Uint8Array): string {
  let out = '';
  for (const b of bytes) out += b.toString(16).padStart(2, '0');
  return out;
}

/**
 * Derives the opaque cookie token for a PIN. HMAC keyed by `secret`, so the
 * token cannot be reversed to the PIN without the secret. When the host has no
 * secret the PIN doubles as the key — the token is then only as strong as the
 * PIN itself (see operations.md, "The cookie is a fingerprint of the PIN").
 */
export async function deriveGateToken(
  pin: string,
  secret: string | undefined
): Promise<string> {
  const key = await crypto.subtle.importKey(
    'raw',
    encoder.encode(secret ?? pin),
    { name: 'HMAC', hash: 'SHA-256' },
    false,
    ['sign']
  );
  // Versioned message: change the prefix to invalidate every issued cookie
  // without touching the PIN.
  const signature = await crypto.subtle.sign(
    'HMAC',
    key,
    encoder.encode(`site-gate:v1:${pin}`)
  );
  return toHex(new Uint8Array(signature));
}

/**
 * Constant-time string comparison. Both the PIN check and the cookie check go
 * through here so that a wrong guess takes the same time whether it fails on
 * the first byte or the last.
 */
export function constantTimeEqual(a: string, b: string): boolean {
  const ab = encoder.encode(a);
  const bb = encoder.encode(b);
  let diff = ab.length ^ bb.length;
  const n = Math.max(ab.length, bb.length);
  for (let i = 0; i < n; i++) diff |= (ab[i] ?? 0) ^ (bb[i] ?? 0);
  return diff === 0;
}

/**
 * Normalises the "return to" path carried by the unlock form. Anything that is
 * not a same-origin path collapses to "/". `startsWith('/')` alone is not a
 * check: `//evil.example`, `/\evil.example` and `///evil.example` all pass it
 * and all resolve to another host.
 */
export function safeReturnPath(
  raw: unknown,
  origin: string,
  unlockPath: string
): string {
  if (typeof raw !== 'string' || raw === '') return '/';
  if (!raw.startsWith('/') || raw.startsWith('//') || raw.startsWith('/\\')) {
    return '/';
  }
  let url: URL;
  try {
    url = new URL(raw, origin);
  } catch {
    return '/';
  }
  if (url.origin !== origin) return '/';
  if (url.pathname === unlockPath) return '/'; // never bounce back into the form
  return url.pathname + url.search; // drops any fragment and credentials
}

const HTML_ESCAPES: Record<string, string> = {
  '&': '&amp;',
  '<': '&lt;',
  '>': '&gt;',
  '"': '&quot;',
  "'": '&#39;',
};

/** Escapes text for use in an HTML attribute or text node. */
export function escapeHtml(value: string): string {
  return value.replace(/[&<>"']/g, ch => HTML_ESCAPES[ch] ?? ch);
}

/**
 * Picks the client key the attempt budget is charged to. Vercel and most
 * reverse proxies set `x-forwarded-for`; the first entry is the client. Behind
 * no proxy every request shares one key, which still caps total guessing.
 */
export function clientKey(headers: Headers): string {
  const forwarded = headers.get('x-forwarded-for');
  const first = forwarded?.split(',')[0]?.trim();
  return first || headers.get('x-real-ip') || 'unknown';
}

/**
 * Picks the gate page language: the app's locale cookie first, then
 * `Accept-Language`, then the default. The proxy runs before any i18n
 * provider exists, so this is the only locale signal available to it.
 */
export function pickLocale(
  headers: Headers,
  cookieLocale: string | undefined,
  supported: readonly string[],
  fallback: string
): string {
  if (cookieLocale && supported.includes(cookieLocale)) return cookieLocale;
  const accept = headers.get('accept-language') ?? '';
  for (const part of accept.split(',')) {
    const tag = part.split(';')[0]?.trim().toLowerCase();
    if (!tag) continue;
    const base = tag.split('-')[0] ?? tag;
    const hit = supported.find(l => l.toLowerCase() === tag || l.toLowerCase() === base);
    if (hit) return hit;
  }
  return fallback;
}
```

### Why HMAC and not a hash

The source derived the cookie as `SHA-256("<salt>:" + PIN)` with the salt in
source code. That is not a secret derivation; it is a fingerprint. A PIN drawn
from a small space (four to six digits is what people type into a numeric
field) is recovered from the cookie by hashing every candidate — ten thousand
SHA-256 calls, a few milliseconds. HMAC keyed by a server-side secret makes
the cookie useless offline: without the secret there is nothing to brute-force
against.

When `SITE_GATE_SECRET` is unset the PIN doubles as the key and the same
weakness returns. The handler warns once per instance; the fix is one env var.

The message is versioned (`site-gate:v1:`). Bumping the version invalidates
every issued cookie without changing the PIN — the same effect as rotating the
secret, but visible in a diff.

### Why a constant-time compare

`pin === SITE_PIN` returns as soon as a character differs. Over a network the
signal is small, but the fix is fifteen lines and removes the question.
Comparing the **digest** of the submitted PIN to the expected digest, rather
than the PINs themselves, gives both sides the same length and hides the PIN's
length too.

### Why `safeReturnPath` resolves against the origin

Every check verified against the WHATWG URL parser:

| Input | `startsWith('/')` | Resolves to |
|---|---|---|
| `/pl/robot?x=1` | yes | `https://site.example/pl/robot?x=1` |
| `//evil.example/x` | yes | `https://evil.example/x` |
| `/\evil.example` | yes | `https://evil.example/` |
| `///evil.example` | yes | `https://evil.example/` |
| `/%2F%2Fevil.example` | yes | `https://site.example/%2F%2Fevil.example` |
| `https://evil.example` | no | `https://evil.example/` |

The attack is not theoretical: a page anywhere can auto-submit a form to
`/__unlock` with a wrong PIN and `next=//evil.example`. The victim sees the
genuine gate on the genuine domain, types the PIN, and lands on the attacker's
copy of the site. Resolve the path, compare `url.origin` to the request's
origin, and return `pathname + search` so a fragment or embedded credentials
are dropped as well.

### Why the client key comes from headers

`request.ip` no longer exists on `NextRequest`. On Vercel and behind most
reverse proxies, `x-forwarded-for` is set by the platform and its first entry
is the client. Behind no proxy at all every client shares the key `unknown`,
which still caps total guessing on that instance. Do not read
`x-forwarded-for` from a request that reached Node directly from the internet:
the client sets it.

## Attempt store

```ts
// file: lib/site-gate/attempts.ts
/**
 * Failed-attempt budget. The in-memory store is per server instance: on a
 * platform that fans requests across instances an attacker gets `limit`
 * guesses per instance per window, not `limit` in total. That still turns
 * unbounded guessing into a slow crawl; for a hard cap, implement this
 * interface over a shared store (Redis, a database row) — see adaptation.md.
 */
export type AttemptStore = {
  /** Records one attempt and returns whether it is within budget. */
  hit(key: string, now?: number): Promise<boolean> | boolean;
  /** Clears the budget after a successful unlock. */
  reset(key: string): Promise<void> | void;
};

export function createMemoryAttemptStore(
  limit: number,
  windowMs: number,
  maxKeys = 10_000
): AttemptStore {
  const store = new Map<string, number[]>();

  function prune(now: number) {
    for (const [key, stamps] of store) {
      const live = stamps.filter(ts => now - ts < windowMs);
      if (live.length === 0) store.delete(key);
      else store.set(key, live);
    }
  }

  return {
    hit(key, now = Date.now()) {
      if (store.size >= maxKeys) prune(now); // bound memory under a key flood
      const live = (store.get(key) ?? []).filter(ts => now - ts < windowMs);
      if (live.length >= limit) {
        store.set(key, live);
        return false;
      }
      live.push(now);
      store.set(key, live);
      return true;
    },
    reset(key) {
      store.delete(key);
    },
  };
}
```

Five attempts per fifteen minutes per client, then `429` with `Retry-After`.
A successful unlock resets the client's budget so a typo does not lock out the
person who knows the PIN. The map is bounded: at `maxKeys` entries it prunes
expired ones before admitting a new key, so a flood of spoofed client keys
cannot grow memory without limit.

Per instance means per instance. A serverless platform running the gate on
many instances multiplies the budget by the instance count. For a hard global
cap, implement the same two-method interface over a shared store — the sketch
is in [adaptation.md](adaptation.md).

## Checklist

- [ ] `SITE_PIN` is eight or more characters; the field accepts any string
- [ ] `SITE_GATE_SECRET` set wherever `SITE_PIN` is
- [ ] Nothing compares the PIN or the cookie with `===`
- [ ] Nothing accepts a return path without `safeReturnPath`
- [ ] The attempt store's limits match the `Retry-After` the handler sends (they share `config.attempts`)
