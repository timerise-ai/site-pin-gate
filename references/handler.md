# The page, the handler, the wiring

The handler is the only file that knows about `NextRequest`. It is created
once per server instance so the expected cookie token is derived once and the
attempt store lives across requests; the source recomputed a digest on every
request and had no store at all.

## Behaviour contract

| Request | Response | Headers |
|---|---|---|
| Any path, `SITE_PIN` unset | `null` — the app runs | — |
| Any path, valid cookie | `null` | — |
| Any path, no or invalid cookie | `401`, gate page with the request's path and query as the return path | `cache-control: no-store`, `x-robots-tag: noindex, nofollow` |
| `GET /__unlock` (any non-POST) | `303` to `/` | — |
| `POST /__unlock`, body is not a form | `400`, gate page, "invalid" message | as 401 |
| `POST /__unlock`, budget exhausted | `429`, gate page, "too many attempts" | as 401, plus `retry-after` in seconds |
| `POST /__unlock`, wrong PIN | `401`, gate page, "wrong" message, return path preserved | as 401 |
| `POST /__unlock`, right PIN | `303` to the sanitised return path | `set-cookie: <name>=<token>; HttpOnly; SameSite=Lax; Secure; Path=/; Max-Age=…` |

Two things the table encodes that are easy to undo by accident:

- **`303`, not the default `307`.** A 307 tells the browser to repeat the
  request *with the same method and body*. After a form POST that means the
  PIN is POSTed again to the landing page — and if the host's locale routing
  then redirects `/` to `/en`, POSTed a third time. `303` converts the follow-up
  to a GET.
- **The budget is charged before the PIN is checked.** Otherwise a client past
  its budget could keep guessing and only be refused after a *correct* guess.

## The gate page

```ts
// file: lib/site-gate/page.ts
import { escapeHtml } from './core';

export type GateStrings = {
  /** <title> of the gate page. */
  title: string;
  /** One line under the brand. */
  prompt: string;
  /** Accessible label of the PIN field. */
  label: string;
  button: string;
  errors: {
    wrong: string;
    rateLimited: string;
    invalid: string;
  };
};

export type GateError = keyof GateStrings['errors'] | null;

export const GATE_STRINGS_EN: GateStrings = {
  title: 'Restricted',
  prompt: 'Enter the access PIN to continue.',
  label: 'Access PIN',
  button: 'Unlock',
  errors: {
    wrong: 'Incorrect PIN. Try again.',
    rateLimited: 'Too many attempts. Wait a few minutes and try again.',
    invalid: 'That request could not be read. Try again.',
  },
};

export type GatePageInput = {
  brand: string;
  lang: string;
  unlockPath: string;
  /** Same-origin path to return to after unlock. Already sanitised. */
  next: string;
  error: GateError;
  strings: GateStrings;
};

/**
 * Renders the gate as a self-contained document. Inline CSS is deliberate: the
 * proxy runs before the app, has no access to its stylesheet or components,
 * and the page must render even when everything behind the gate is broken.
 * Restyle through the custom properties in `:root`; the `data-site-gate`
 * attributes are stable hooks if the host serves its own stylesheet.
 */
export function renderGatePage(input: GatePageInput): string {
  const { brand, lang, unlockPath, next, error, strings } = input;
  const message = error ? strings.errors[error] : '';
  return `<!doctype html>
<html lang="${escapeHtml(lang)}">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<meta name="robots" content="noindex, nofollow" />
<title>${escapeHtml(brand)} — ${escapeHtml(strings.title)}</title>
<style>
  :root {
    color-scheme: light dark;
    --gate-bg: #f5f6f8; --gate-fg: #111418; --gate-muted: #5f6b76;
    --gate-card: #ffffff; --gate-border: rgba(0,0,0,0.12);
    --gate-accent: #2563eb; --gate-error: #b91c1c;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --gate-bg: #0b0f14; --gate-fg: #e6edf3; --gate-muted: #8b98a5;
      --gate-card: rgba(255,255,255,0.03); --gate-border: rgba(255,255,255,0.1);
      --gate-accent: #3b82f6; --gate-error: #f87171;
    }
  }
  * { box-sizing: border-box; }
  body {
    margin: 0; min-height: 100vh; display: grid; place-items: center;
    font-family: ui-sans-serif, system-ui, sans-serif;
    background: var(--gate-bg); color: var(--gate-fg);
  }
  form {
    width: min(92vw, 360px); padding: 2rem; border-radius: 16px;
    background: var(--gate-card); border: 1px solid var(--gate-border);
    text-align: center;
  }
  h1 { font-size: 0.95rem; font-weight: 600; letter-spacing: 0.14em;
       text-transform: uppercase; color: var(--gate-muted); margin: 0 0 0.35rem; }
  p { margin: 0 0 1.5rem; font-size: 0.85rem; color: var(--gate-muted); }
  input[type=password] {
    width: 100%; padding: 0.85rem 1rem; text-align: center; font-size: 1.4rem;
    letter-spacing: 0.4em; border-radius: 10px; border: 1px solid var(--gate-border);
    background: transparent; color: var(--gate-fg); outline: none;
  }
  input[type=password]:focus { border-color: var(--gate-accent); }
  button {
    width: 100%; margin-top: 1rem; padding: 0.8rem 1rem; font-size: 0.95rem;
    font-weight: 600; border: none; border-radius: 10px; cursor: pointer;
    background: var(--gate-accent); color: #fff;
  }
  [data-site-gate="error"] { color: var(--gate-error); font-size: 0.8rem;
    margin: 0.75rem 0 0; min-height: 1rem; }
</style>
</head>
<body>
  <form method="POST" action="${escapeHtml(unlockPath)}" data-site-gate="form">
    <h1>${escapeHtml(brand)}</h1>
    <p>${escapeHtml(strings.prompt)}</p>
    <input type="password" name="pin" autocomplete="one-time-code" autofocus
           required maxlength="128" aria-label="${escapeHtml(strings.label)}"
           ${error ? 'aria-invalid="true" aria-describedby="gate-error"' : ''} />
    <input type="hidden" name="next" value="${escapeHtml(next)}" />
    <button type="submit">${escapeHtml(strings.button)}</button>
    <p id="gate-error" data-site-gate="error" role="alert">${escapeHtml(message)}</p>
  </form>
</body>
</html>`;
}
```

Notes on the markup:

- `type="password"` with `autocomplete="one-time-code"` keeps the PIN out of
  the password manager's save prompt while letting iOS offer a pasted code.
  The source used a text field with `inputmode="numeric"`, which told every
  visitor the PIN was digits.
- The error paragraph has `role="alert"` and is referenced by
  `aria-describedby` when present, so a screen reader announces the failure.
- Everything interpolated goes through `escapeHtml` — brand, strings, the
  return path, the unlock path. The source escaped only `"` in the return
  path. In an attribute that happened to be enough; the cost of doing it
  properly is one function.
- `color-scheme: light dark` plus two palettes. The page follows the visitor's
  system setting because it has no way to know the app's theme.

## The handler

```ts
// file: lib/site-gate/handler.ts
import { type NextRequest, NextResponse } from 'next/server';

import { type AttemptStore, createMemoryAttemptStore } from './attempts';
import { readSiteGateConfig, type SiteGateConfig } from './config';
import {
  clientKey,
  constantTimeEqual,
  deriveGateToken,
  pickLocale,
  safeReturnPath,
} from './core';
import {
  GATE_STRINGS_EN,
  type GateError,
  type GateStrings,
  renderGatePage,
} from './page';

export type SiteGateDeps = {
  config: SiteGateConfig;
  attempts: AttemptStore;
  /** Locales the gate page can speak; keys of `strings`. */
  strings: Record<string, GateStrings>;
  defaultLocale: string;
  /** Cookie the host's i18n layer uses to remember the locale, if any. */
  localeCookieName?: string;
  log: Pick<Console, 'warn'>;
};

const MAX_PIN_LENGTH = 128;

/** Builds the default dependency set once per server instance. */
export function createSiteGate(overrides: Partial<SiteGateDeps> = {}) {
  const config = overrides.config ?? readSiteGateConfig();
  const deps: SiteGateDeps = {
    config,
    attempts:
      overrides.attempts ??
      createMemoryAttemptStore(
        config.attempts.limit,
        config.attempts.windowMs
      ),
    strings: overrides.strings ?? { en: GATE_STRINGS_EN },
    defaultLocale: overrides.defaultLocale ?? 'en',
    localeCookieName: overrides.localeCookieName,
    log: overrides.log ?? console,
  };

  // The expected token depends only on config, so derive it once and reuse
  // the promise; the source recomputed a digest on every request.
  let expectedToken: Promise<string> | undefined;
  let warnedWeakCookie = false;

  function expected(): Promise<string> {
    if (!expectedToken) {
      expectedToken = deriveGateToken(config.pin ?? '', config.secret);
    }
    return expectedToken;
  }

  function gatePage(
    request: NextRequest,
    next: string,
    error: GateError,
    status: number,
    extraHeaders: Record<string, string> = {}
  ): NextResponse {
    const cookieLocale = deps.localeCookieName
      ? request.cookies.get(deps.localeCookieName)?.value
      : undefined;
    const lang = pickLocale(
      request.headers,
      cookieLocale,
      Object.keys(deps.strings),
      deps.defaultLocale
    );
    const strings = deps.strings[lang] ?? deps.strings[deps.defaultLocale];
    const html = renderGatePage({
      brand: config.brand,
      lang,
      unlockPath: config.unlockPath,
      next,
      error,
      strings: strings ?? GATE_STRINGS_EN,
    });
    return new NextResponse(html, {
      status,
      headers: {
        'content-type': 'text/html; charset=utf-8',
        'cache-control': 'no-store',
        'x-robots-tag': 'noindex, nofollow',
        ...extraHeaders,
      },
    });
  }

  /**
   * Returns a response to send instead of the app (the gate page, a redirect),
   * or `null` when the request may proceed.
   */
  return async function siteGate(
    request: NextRequest
  ): Promise<NextResponse | null> {
    if (!config.pin) return null; // gate disabled

    if (!config.secret && !warnedWeakCookie) {
      warnedWeakCookie = true;
      deps.log.warn(
        '[site-gate] SITE_GATE_SECRET is unset: the access cookie is a fingerprint of the PIN.'
      );
    }

    const { pathname, search, origin } = request.nextUrl;

    if (pathname !== config.unlockPath) {
      const token = request.cookies.get(config.cookieName)?.value;
      if (token && constantTimeEqual(token, await expected())) return null;
      return gatePage(request, pathname + search, null, 401);
    }

    // --- the unlock form ---------------------------------------------------
    if (request.method !== 'POST') {
      return NextResponse.redirect(new URL('/', origin), 303);
    }

    let pin = '';
    let next = '/';
    try {
      const form = await request.formData();
      pin = String(form.get('pin') ?? '').trim();
      next = safeReturnPath(form.get('next'), origin, config.unlockPath);
    } catch {
      // Not a form body (JSON, empty, wrong content-type). The source let
      // this throw and answered with a 500 page.
      return gatePage(request, '/', 'invalid', 400);
    }

    const key = clientKey(request.headers);
    if (!(await deps.attempts.hit(key))) {
      deps.log.warn(`[site-gate] attempt budget exhausted for ${key}`);
      return gatePage(request, next, 'rateLimited', 429, {
        'retry-after': String(Math.ceil(config.attempts.windowMs / 1000)),
      });
    }

    const submitted =
      pin.length > 0 && pin.length <= MAX_PIN_LENGTH
        ? await deriveGateToken(pin, config.secret)
        : '';
    if (!submitted || !constantTimeEqual(submitted, await expected())) {
      deps.log.warn(`[site-gate] wrong PIN from ${key}`);
      return gatePage(request, next, 'wrong', 401);
    }

    await deps.attempts.reset(key);
    // 303 turns the browser's POST into a GET. With the default 307 the
    // browser re-POSTs the form — PIN included — to the page it lands on.
    const response = NextResponse.redirect(new URL(next, origin), 303);
    response.cookies.set(config.cookieName, await expected(), {
      httpOnly: true,
      sameSite: 'lax',
      secure: config.secureCookie,
      path: '/',
      maxAge: config.cookieMaxAgeSeconds,
    });
    return response;
  };
}
```

The dependency object exists so tests can inject a config, a store and a
silent logger, and so a host can swap the store for a shared one. Nothing in
it is meant to change per request.

`request.nextUrl.origin` is the origin the request arrived on as Next sees it,
which behind a platform proxy is the public origin. `safeReturnPath` compares
against it, and the redirect is built from it, so the redirect never points at
an internal host name.

## Wiring

```ts
// file: proxy.ts
import { type NextRequest, NextResponse } from 'next/server';

import { createSiteGate } from '@/lib/site-gate/handler';

// Built once per server instance: the expected token and the attempt store
// live in this closure.
const siteGate = createSiteGate();

// Next.js 16 `proxy` convention. On Next.js 13–15 name the file
// `middleware.ts` and export `middleware` instead; the body is identical.
export async function proxy(request: NextRequest) {
  const gate = await siteGate(request);
  if (gate) return gate;
  return NextResponse.next(); // or hand off to the host's own proxy logic
}

export const config = {
  // Everything except Next's build assets. See adaptation.md for the trade-off
  // between this and a pages-only matcher.
  matcher: ['/((?!_next/static|_next/image|favicon\\.ico).*)'],
};
```

For Next 13–15, name the file `middleware.ts` and export `middleware`. To run
the gate ahead of an existing proxy (locale routing, auth, rewrites), call it
first and return its response when it gives one — the composed example is in
[adaptation.md](adaptation.md).

The `config.matcher` line is the boundary. Read "Choosing the matcher" in
[adaptation.md](adaptation.md) before changing it; the pages-only matcher the
source used left the sitemap, the OG images and the API public.

## What a locked client-side navigation does

Once unlocked, the cookie expires after thirty days. If it expires mid-session
the next client-side navigation fetches an RSC payload and receives the gate
page instead. Next's router treats a non-RSC response as a signal to perform a
full navigation to that URL, so the visitor sees the gate rather than a broken
transition. Not observed under test — stated from the router's documented
behaviour for middleware responses.

## Checklist

- [ ] `createSiteGate()` is called at module scope, not inside the handler
- [ ] The gate runs before any other logic in the proxy
- [ ] The unlock redirect is `303`
- [ ] The matcher excludes only what must be public, by name
- [ ] `localeCookieName` and `strings` passed when the host has locales
