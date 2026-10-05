# Provenance

## Sources

| | Source | What it contributed |
|---|---|---|
| A | A temporary gate on a multi-locale marketing site, audited before the skill was written | The architecture, and the twelve-entry ledger below |
| B | The skill applied, from 0.3.1, to the staging site of a multi-vendor marketplace: an existing proxy doing session refresh and CSP, nine UI locales including Arabic, and inbound webhooks from payment, KYC, email, tracking, video and helpdesk providers | The locale resolver and text-direction seams, the bypass list, the machine-caller inventory, and the operations notes in *Applied to a second host* below |

Unless an entry says otherwise, it comes from source A.

## Source A

The earlier implementation this module was audited against was a temporary
site-wide PIN gate on a multi-locale App Router marketing site, kept private
ahead of its launch: a self-contained block in `proxy.ts`, run before locale
routing, with a `SITE_PIN` env var, a cookie holding a SHA-256 of the PIN, and
an inline HTML form posting to `/__unlock`. The architecture here is that
block's. The templates are not a transcription of it, and the ledger below is
why.

Every entry marked **verified live** was reproduced with `curl` against that
implementation running in a dev server with `SITE_PIN=1234`. The rest were
confirmed by reading it and, where a parser is involved, by running the input
through the parser.

## Fixed in the templates

### 1. The unlock form redirected off-site (verified live)

The return path was accepted if it started with `/`. `//evil.example/x`
starts with `/`. Posting the correct PIN with that as `next` produced
`location: http://evil.example/x`; `/\evil.example` and `///evil.example`
behave the same in the URL parser. A page anywhere could auto-submit a wrong
PIN with a hostile `next`, let the victim see the real gate on the real
domain, and collect them on the other side.

**Shipped:** `safeReturnPath` resolves the value against the request origin
and compares origins. See [module.md](module.md).

**Added in 0.3.6:** the resolved path is resolved once more before it is
returned. Dot segments normalise `/a/..//evil.example` to the pathname
`//evil.example`, which passed every earlier check and which the unlock
redirect resolved to `https://evil.example/`. An agent eval against 0.3.5
found it; the suite now carries two dot-segment cases, and the handler test
posts one with the right PIN.

### 2. The cookie was the PIN in disguise (verified live)

The cookie held `SHA-256("<constant>:" + PIN)`, with the constant in the code.
Numeric PINs are what a field marked `inputmode="numeric"` invites. A
ten-thousand-iteration loop recovered the PIN from a captured cookie in
milliseconds. The comment above the code said the raw PIN "never sits in the
browser cookie", which was true and beside the point.

**Shipped:** HMAC-SHA-256 keyed by `SITE_GATE_SECRET`; a warning when the
secret is absent. See [module.md](module.md).

### 3. A successful unlock re-POSTed the PIN to the landing page (verified live)

`NextResponse.redirect` defaults to `307`, which preserves method and body.
The dev server log showed `POST /pl/robot 200` after the unlock: the page
rendered from a POST carrying `pin=1234` in its body, and with a locale
redirect chain the body travels once more. Reloading the landing page in a
browser prompts to resubmit the form.

**Shipped:** `303`. See [handler.md](handler.md).

### 4. A non-form body crashed the proxy (verified live)

`request.formData()` throws a `TypeError` for any body that is not
`multipart/form-data` or `application/x-www-form-urlencoded`. It was not
caught; a `POST` with a JSON body returned a 500 page.

**Shipped:** the parse is wrapped; a bad body is a `400` with the gate page.
See [handler.md](handler.md).

### 5. No attempt budget

Nothing counted failures. A four-digit PIN is ten thousand requests to a
handler that costs a hash each; a laptop does that in minutes. The host app
had a rate-limit helper used by its API routes; the gate did not call it.

**Shipped:** a per-client sliding-window budget with `429` and `Retry-After`,
reset on success, memory-bounded; a seam for a shared store. See
[module.md](module.md) and [adaptation.md](adaptation.md).

### 6. Plain `===` on the PIN and on the cookie

Both comparisons short-circuit on the first differing character.

**Shipped:** `constantTimeEqual` over equal-length digests.

### 7. The pages-only matcher published the route list (verified live)

The matcher excluded `/api` and every path with a dot. Without a cookie,
`/sitemap.xml` returned 200 and listed every URL of the unlaunched site; the
OG image and the API routes were reachable as well.

**Shipped:** the default matcher excludes build assets only, and the
pages-only shape is documented as a choice with its consequences. See
[adaptation.md](adaptation.md).

### 8. The digest was recomputed on every request

`gateToken(SITE_PIN)` ran once per request for a value that never changes.

**Shipped:** derived once per instance in `createSiteGate`.

### 9. One language, brand hardcoded, on a multi-locale site

The gate page carried the site's brand as a literal and spoke one language to
every visitor, whose locale cookie was right there in the request.

**Shipped:** a strings table per locale, chosen from the locale cookie and
`Accept-Language`; brand from an env var. See [adaptation.md](adaptation.md).

### 10. A whitespace PIN armed a gate nobody could open

`SITE_PIN=" "` passed the truthiness check, but submitted PINs were trimmed,
so nothing ever matched.

**Shipped:** the env value is trimmed and blank means off.

### 11. Return path escaped for `"` only

The hidden field escaped double quotes and nothing else. In a double-quoted
attribute that is enough to stay inside the attribute, so no exploit was
found; it is still one function away from correct.

**Shipped:** `escapeHtml` on everything interpolated.

### 12. `status: error ? 401 : 401`

Cosmetic; both branches were 401. Recorded because it is the kind of line
that gets "fixed" into a wrong status. 401 on both is right.

## Kept deliberately

- **The gate lives in the proxy, not in a layout.** A layout runs after the
  request has been routed and cannot stop a route handler, an RSC fetch or a
  statically served page. The proxy sees every matched request first.
- **401 for the gate page.** It keeps every crawler out and matches what
  hosting platforms return for protected deployments. A `200` with a form
  would be indexed as the site's content.
- **Inline CSS, one document, no external references.** The proxy has no
  stylesheet, no components and no i18n provider, and the page must render
  when the app behind it is broken.
- **The cookie token is derived from the PIN, with no session store.**
  Rotating the PIN is the logout-everyone mechanism, and there is nothing to
  clean up.
- **`SameSite=Lax`.** `Strict` would drop the cookie on every arrival from a
  link in Slack or email, showing the gate again to someone already unlocked.
- **No CSRF token on the form.** There is no per-user state to hijack; a
  forged unlock with the right PIN gains the attacker nothing they do not
  already have.
- **`inputmode="numeric"` is gone, `type="password"` is in.** A numeric input
  mode advertises the PIN's alphabet; a password field does not.

## Added

Designed in the skill and never run in the earlier implementation, all marked
as additions above: the attempt budget and `429`, the keyed token and
`SITE_GATE_SECRET`, locale selection and the strings table, the return-path
sanitiser, `400` on a bad body, `x-robots-tag`, `no-store` on the unlock
redirects, `Retry-After`, the warn-level
log lines, the brand env var, and the test suite. The Redis attempt store, the
lock endpoint, per-client PINs and the bypass header in
[operations.md](operations.md) are designs only.

From source B, built and smoke-tested there but not yet run armed: the
locale resolver and text-direction seams, the bypass list, and the browser-run
unlock in [operations.md](operations.md), which is a sketch.

## Applied to a second host (source B)

Source B was built from the templates rather than audited, so what it adds
is what the templates did not anticipate, not defects in an earlier gate. It
was smoke-tested on a production build with a throwaway PIN before its own
environment was armed; the results are marked.

- **Its proxy already had a matcher, and other jobs on it.** Session refresh
  and a CSP nonce ran on almost every path, webhooks included. Replacing that
  matcher to suit the gate would have broken them, so the gate was skipped in
  code for a list of public prefixes. **Shipped:** the bypass-list variant and
  its edge rule in [adaptation.md](adaptation.md). **Verified on B's build:**
  pages answered `401`; `/healthz` and an API route `200`; an unsigned payment
  webhook reached its route and got the route's own signature error.
- **The reason for the gate was the webhooks.** The client asked for platform
  password protection; that would have refused every provider callback and
  every cron trigger called by URL, each needing a bypass secret re-registered
  with the provider. **Shipped:** the comparison row and paragraph in
  [operations.md](operations.md), and the machine-caller inventory in the host
  probe.
- **`pickLocale` could not reach a script-subtagged locale.** `zh-CN` has base
  `zh`, which never matches `zh-Hans`, and the page had no text direction for
  Arabic. B already had a negotiation function with those aliases. **Shipped:**
  `resolveLocale` and `localeDir` on `createSiteGate`, `dir` on the page, and
  two handler tests. **Verified on B's build:** `NEXT_LOCALE=ar` rendered
  `<html lang="ar" dir="rtl">`. Kept: `pickLocale` stays the default for hosts
  without their own negotiation.
- **`translate="no"` on the page and `dir="ltr"` on the PIN field.** B's app
  already opted out of browser translation; a gate page offered for
  translation, or a PIN field laid out right to left, is noise. **Shipped**
  in the page template.
- **The host's test runner defaulted to `jsdom`.** The suites need Node for
  `NextRequest` and `crypto.subtle`. **Shipped:** the
  `// @vitest-environment node` note in [testing.md](testing.md).
- **Staging was the *Production* environment of a separate project.** Setting
  the variables on *Preview* would have armed nothing. **Shipped:** a
  paragraph in *Arming it, per environment*.
- **B was built from a stale installed copy.** The installed skill was 0.3.1
  while the repository was at 0.3.7, so B shipped without the 0.3.3 key-cap
  eviction, the 0.3.4 `no-store` on the unlock redirects and the 0.3.6
  dot-segment fix to `safeReturnPath`: a live open redirect, closed before the
  PIN was set. The ported tests failed on B's code and passed after.
  **Shipped:** *An app built from an older copy of this skill* in
  [operations.md](operations.md), and item 0 of the fix order below.

## If you are upgrading an existing gate

Fix order, most damaging first:

0. If the gate was built from this skill, read the changelog's *Security* and
   *Fixed* entries since that version first; a gate built before 0.3.6 has
   the dot-segment redirect.
1. Sanitise the return path (defect 1): one function, closes a live
   phishing vector.
2. Change the unlock redirect to `303` (defect 3).
3. Key the cookie token with a secret (defect 2): invalidates every existing
   cookie, so schedule it.
4. Wrap `formData()` (defect 4).
5. Add the attempt budget (defect 5), reusing the host's limiter if it has one.
6. Constant-time compares (defect 6).
7. Decide the matcher on purpose (defect 7).
