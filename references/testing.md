# Testing

Two files. The first exercises every pure helper and the attempt store. The
second drives the handler with real `NextRequest` objects through every row of
the behaviour contract in [handler.md](handler.md), including the three
off-origin return paths the sanitiser has to collapse.

Both files import from `vitest`. `bun test` rewrites that import to its own
runner, so they run unchanged under either:

```bash
npx vitest run lib/site-gate      # vitest
bun test lib/site-gate            # bun
```

They were verified under `bun test` and `vitest run` (40 tests) and
type-checked under `strict` and `noUncheckedIndexedAccess`. `next/server` must
be resolvable, and it is, in any Next.js app.

A host with no test runner gets `vitest` as a dev dependency and a
`"test": "vitest run"` script: run `npm i -D vitest`. The package registry is
not an external service in the sense of "no external services are reachable";
that phrase is about the app's runtime credentials. Do not pass `--offline`,
search the npm cache for a copy, rewrite the `vitest` import, convert the
assertions to `node:assert` or hand-roll a runner: the suite is the behaviour
contract, and it should run as written. If the install genuinely fails, run
`bun test` when bun is present, and otherwise report the tests as not run.

A host whose vitest config defaults to `jsdom` (common when it tests React
components) runs these files in the wrong environment: `NextRequest`, form
parsing and `crypto.subtle` belong to Node. Make `// @vitest-environment node`
the first line of each file, above the `// file:` comment, rather than
changing the host's default or moving the files. Under `bun test` the line is
an ordinary comment.

## Helpers and store

```ts
// file: lib/site-gate/core.test.ts
import { describe, expect, it } from 'vitest';

import { createMemoryAttemptStore } from './attempts';
import {
  clientKey,
  constantTimeEqual,
  deriveGateToken,
  escapeHtml,
  pickLocale,
  safeReturnPath,
} from './core';

const ORIGIN = 'https://site.example';
const UNLOCK = '/__unlock';

describe('safeReturnPath', () => {
  it('keeps same-origin paths with their query', () => {
    expect(safeReturnPath('/pl/robot?x=1', ORIGIN, UNLOCK)).toBe('/pl/robot?x=1');
  });
  it('drops the fragment', () => {
    expect(safeReturnPath('/docs#top', ORIGIN, UNLOCK)).toBe('/docs');
  });
  it.each([
    '//evil.example/x',
    '/\\evil.example',
    '///evil.example',
    '/a/..//evil.example',
    '/.//evil.example/x',
    'https://evil.example',
    'javascript:alert(1)',
    '',
    undefined,
    null,
    42,
  ])('collapses %j to /', raw => {
    expect(safeReturnPath(raw, ORIGIN, UNLOCK)).toBe('/');
  });
  it('never returns to the unlock path itself', () => {
    expect(safeReturnPath(UNLOCK, ORIGIN, UNLOCK)).toBe('/');
  });
  it('keeps a percent-encoded double slash, which stays same-origin', () => {
    expect(safeReturnPath('/%2F%2Fevil.example', ORIGIN, UNLOCK)).toBe('/%2F%2Fevil.example');
  });
});

describe('escapeHtml', () => {
  it('escapes the five HTML metacharacters', () => {
    expect(escapeHtml(`<a href="x">&'`)).toBe('&lt;a href=&quot;x&quot;&gt;&amp;&#39;');
  });
});

describe('constantTimeEqual', () => {
  it('compares equal and unequal strings', () => {
    expect(constantTimeEqual('abc', 'abc')).toBe(true);
    expect(constantTimeEqual('abc', 'abd')).toBe(false);
    expect(constantTimeEqual('abc', 'abcd')).toBe(false);
    expect(constantTimeEqual('', '')).toBe(true);
  });
});

describe('deriveGateToken', () => {
  it('is deterministic and hex', async () => {
    const a = await deriveGateToken('1234', 'secret');
    expect(a).toBe(await deriveGateToken('1234', 'secret'));
    expect(a).toMatch(/^[0-9a-f]{64}$/);
  });
  it('changes with the PIN and with the secret', async () => {
    const base = await deriveGateToken('1234', 'secret');
    expect(await deriveGateToken('1235', 'secret')).not.toBe(base);
    expect(await deriveGateToken('1234', 'other')).not.toBe(base);
    expect(await deriveGateToken('1234', undefined)).not.toBe(base);
  });
});

describe('createMemoryAttemptStore', () => {
  it('allows `limit` hits per window, then refuses, then recovers', () => {
    const store = createMemoryAttemptStore(3, 1000);
    const t0 = 1_000_000;
    expect(store.hit('ip', t0)).toBe(true);
    expect(store.hit('ip', t0 + 1)).toBe(true);
    expect(store.hit('ip', t0 + 2)).toBe(true);
    expect(store.hit('ip', t0 + 3)).toBe(false);
    expect(store.hit('other', t0 + 3)).toBe(true); // budgets are per key
    expect(store.hit('ip', t0 + 1001)).toBe(true); // first hit aged out
  });
  it('reset clears the budget', () => {
    const store = createMemoryAttemptStore(1, 1000);
    expect(store.hit('ip', 0)).toBe(true);
    expect(store.hit('ip', 1)).toBe(false);
    store.reset('ip');
    expect(store.hit('ip', 2)).toBe(true);
  });
  it('prunes expired keys when the key cap is reached', () => {
    const store = createMemoryAttemptStore(1, 100, 2);
    store.hit('a', 0);
    store.hit('b', 0);
    expect(store.hit('c', 500)).toBe(true); // a and b pruned, c admitted
    expect(store.hit('a', 501)).toBe(true); // a was forgotten, so it is fresh
  });
  it('evicts the oldest key when every key is still live', () => {
    const store = createMemoryAttemptStore(1, 1000, 2);
    store.hit('a', 0);
    store.hit('b', 1);
    expect(store.hit('c', 2)).toBe(true); // a evicted, c admitted
    expect(store.hit('b', 3)).toBe(false); // b kept, still spent
    expect(store.hit('a', 4)).toBe(true); // a was forgotten, so it is fresh
  });
});

describe('clientKey', () => {
  it('prefers the first x-forwarded-for entry', () => {
    const h = new Headers({ 'x-forwarded-for': ' 1.2.3.4, 10.0.0.1', 'x-real-ip': '9.9.9.9' });
    expect(clientKey(h)).toBe('1.2.3.4');
  });
  it('falls back to x-real-ip, then a shared key', () => {
    expect(clientKey(new Headers({ 'x-real-ip': '9.9.9.9' }))).toBe('9.9.9.9');
    expect(clientKey(new Headers())).toBe('unknown');
  });
});

describe('pickLocale', () => {
  const supported = ['en', 'pl'] as const;
  it('prefers the locale cookie when supported', () => {
    expect(pickLocale(new Headers({ 'accept-language': 'en' }), 'pl', supported, 'en')).toBe('pl');
  });
  it('reads Accept-Language with regions and q-values', () => {
    const h = new Headers({ 'accept-language': 'de-DE,pl-PL;q=0.8,en;q=0.5' });
    expect(pickLocale(h, undefined, supported, 'en')).toBe('pl');
  });
  it('falls back to the default', () => {
    expect(pickLocale(new Headers(), 'fr', supported, 'en')).toBe('en');
  });
});
```

## Handler

```ts
// file: lib/site-gate/handler.test.ts
import { NextRequest } from 'next/server';
import { describe, expect, it } from 'vitest';

import { readSiteGateConfig } from './config';
import { deriveGateToken } from './core';
import { createSiteGate } from './handler';
import { GATE_STRINGS_EN, renderGatePage } from './page';

const ORIGIN = 'https://site.example';
const silent = { warn: () => {} };

function gate(env: Record<string, string | undefined>, limit = 5) {
  const config = readSiteGateConfig(
    { NODE_ENV: 'production', ...env },
    { attempts: { limit, windowMs: 60_000 } }
  );
  return createSiteGate({ config, log: silent });
}

function get(path: string, init: { cookie?: string; headers?: Record<string, string> } = {}) {
  const headers = new Headers(init.headers);
  if (init.cookie) headers.set('cookie', init.cookie);
  return new NextRequest(ORIGIN + path, { headers });
}

function post(body: string, init: { contentType?: string; ip?: string } = {}) {
  const headers = new Headers({
    'content-type': init.contentType ?? 'application/x-www-form-urlencoded',
  });
  if (init.ip) headers.set('x-forwarded-for', init.ip);
  return new NextRequest(ORIGIN + '/__unlock', { method: 'POST', body, headers });
}

describe('siteGate', () => {
  it('is a no-op when the PIN is unset or blank', async () => {
    expect(await gate({})(get('/'))).toBeNull();
    expect(await gate({ SITE_PIN: '  ' })(get('/'))).toBeNull();
  });

  it('answers a locked request with an uncacheable, unindexable 401 page', async () => {
    const res = await gate({ SITE_PIN: '1234' })(get('/pl/robot?x=1'));
    expect(res?.status).toBe(401);
    expect(res?.headers.get('cache-control')).toBe('no-store');
    expect(res?.headers.get('x-robots-tag')).toBe('noindex, nofollow');
    const html = await res?.text();
    expect(html).toContain('name="next" value="/pl/robot?x=1"');
    expect(html).toContain('<html lang="en" dir="ltr"');
  });

  it('escapes everything it reflects into the form', () => {
    const html = renderGatePage({
      brand: 'Acme <b>',
      lang: 'en',
      dir: 'ltr',
      unlockPath: '/__unlock',
      next: '/a?q="><script>',
      error: null,
      strings: GATE_STRINGS_EN,
    });
    expect(html).not.toContain('<script>');
    expect(html).toContain('value="/a?q=&quot;&gt;&lt;script&gt;"');
    expect(html).toContain('<h1>Acme &lt;b&gt;</h1>');
  });

  it('unlocks with the right PIN: 303, cookie, same-origin redirect', async () => {
    const g = gate({ SITE_PIN: '1234', SITE_GATE_SECRET: 's3cret' });
    const res = await g(post('pin=1234&next=/pl/robot'));
    expect(res?.status).toBe(303);
    expect(res?.headers.get('location')).toBe(ORIGIN + '/pl/robot');
    expect(res?.headers.get('cache-control')).toBe('no-store');
    const cookie = res?.headers.get('set-cookie') ?? '';
    expect(cookie).toContain('site_access=' + (await deriveGateToken('1234', 's3cret')));
    expect(cookie).toMatch(/HttpOnly/i);
    expect(cookie).toMatch(/Secure/i);
    expect(cookie).toMatch(/SameSite=lax/i);
  });

  it('never redirects off-origin after unlock', async () => {
    const g = gate({ SITE_PIN: '1234' });
    for (const next of ['//evil.example/x', '/\\evil.example', '/a/..//evil.example', 'https://evil.example']) {
      const res = await g(post(`pin=1234&next=${encodeURIComponent(next)}`));
      expect(res?.headers.get('location')).toBe(ORIGIN + '/');
    }
  });

  it('lets a valid cookie through and rejects a forged one', async () => {
    const g = gate({ SITE_PIN: '1234', SITE_GATE_SECRET: 's3cret' });
    const token = await deriveGateToken('1234', 's3cret');
    expect(await g(get('/', { cookie: `site_access=${token}` }))).toBeNull();
    const forged = await deriveGateToken('1234', undefined);
    expect((await g(get('/', { cookie: `site_access=${forged}` })))?.status).toBe(401);
  });

  it('rejects a wrong PIN with 401 and keeps the return path', async () => {
    const res = await gate({ SITE_PIN: '1234' })(post('pin=0000&next=/pl/robot'));
    expect(res?.status).toBe(401);
    expect(await res?.text()).toContain('value="/pl/robot"');
  });

  it('answers a non-form body with 400 instead of throwing', async () => {
    const res = await gate({ SITE_PIN: '1234' })(post('{}', { contentType: 'application/json' }));
    expect(res?.status).toBe(400);
  });

  it('redirects GET on the unlock path', async () => {
    const res = await gate({ SITE_PIN: '1234' })(get('/__unlock'));
    expect(res?.status).toBe(303);
    expect(res?.headers.get('location')).toBe(ORIGIN + '/');
    expect(res?.headers.get('cache-control')).toBe('no-store');
  });

  it('speaks the locale the host resolver picks, in its text direction', async () => {
    const g = createSiteGate({
      config: readSiteGateConfig({ SITE_PIN: '1234' }),
      strings: { en: GATE_STRINGS_EN, ar: { ...GATE_STRINGS_EN, button: 'فتح' } },
      resolveLocale: () => 'ar',
      localeDir: locale => (locale === 'ar' ? 'rtl' : 'ltr'),
      log: silent,
    });
    const html = (await (await g(get('/')))?.text()) ?? '';
    expect(html).toContain('<html lang="ar" dir="rtl"');
    expect(html).toContain('فتح');
    expect(html).toMatch(/name="pin"[^>]*dir="ltr"/); // the PIN is typed as written
  });

  it('falls back to the default locale when the resolver picks one it lacks', async () => {
    const g = createSiteGate({
      config: readSiteGateConfig({ SITE_PIN: '1234' }),
      resolveLocale: () => 'de',
      log: silent,
    });
    expect(await (await g(get('/')))?.text()).toContain('<html lang="en" dir="ltr"');
  });

  it('exhausts the attempt budget per client and resets it on success', async () => {
    const g = gate({ SITE_PIN: '1234' }, 2);
    expect((await g(post('pin=0', { ip: '1.1.1.1' })))?.status).toBe(401);
    expect((await g(post('pin=0', { ip: '1.1.1.1' })))?.status).toBe(401);
    const blocked = await g(post('pin=1234', { ip: '1.1.1.1' }));
    expect(blocked?.status).toBe(429); // even the right PIN is refused now
    expect(blocked?.headers.get('retry-after')).toBe('60');
    expect((await g(post('pin=1234', { ip: '2.2.2.2' })))?.status).toBe(303);
  });
});
```

## What is not covered

- The proxy runtime itself. The tests call the handler directly; the wiring in
  `proxy.ts` is four lines, and `crypto.subtle` is available in the Next 16
  proxy. Verify the deployed environment with the smoke checks in
  [operations.md](operations.md).
- Timing. `constantTimeEqual` is tested for correctness, not for timing.
- The Redis attempt store sketch in [adaptation.md](adaptation.md).

## Checklist

- [ ] Both files copied next to the module
- [ ] Runner resolves `next/server` (it runs inside the app, not a scratch dir)
- [ ] The off-origin cases, dot segments included, stay in the suite when the
      sanitiser is touched
- [ ] A host on `jsdom` by default runs both files with `// @vitest-environment node`
- [ ] A bypass list, if the host needed one, has its own test at its edges
