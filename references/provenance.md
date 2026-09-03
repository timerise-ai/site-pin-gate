# Provenance

Extracted from the temporary site-wide PIN gate of a production Next.js 16
marketing site — a bilingual (English, Polish) App Router app with next-intl
locale routing, kept private ahead of launch. The gate was a self-contained
block of about a hundred lines in `proxy.ts`, run before locale routing: a
`SITE_PIN` env var, a cookie holding a SHA-256 of the PIN, an inline HTML
form posting to `/__unlock`. The architecture here is that block's. The
templates are not a transcription of it — the audit below is why.

Every defect marked **verified live** was reproduced with `curl` against the
source running in a dev server with `SITE_PIN=1234`. The rest were confirmed
by reading the source and, where a parser is involved, by running the input
through it.

## Fixed in the templates

### 1. The unlock form redirected off-site — verified live

The return path was accepted if it started with `/`. `//evil.example/x`
starts with `/`. Posting the correct PIN with that as `next` produced
`location: http://evil.example/x`; `/\evil.example` and `///evil.example`
behave the same in the URL parser. A page anywhere could auto-submit a wrong
PIN with a hostile `next`, let the victim see the real gate on the real
domain, and collect them on the other side.

**Shipped:** `safeReturnPath` resolves the value against the request origin
and compares origins. See [module.md](module.md).

### 2. The cookie was the PIN in disguise — verified live

The cookie held `SHA-256("<constant>:" + PIN)`, with the constant in source.
Numeric PINs are what a field marked `inputmode="numeric"` invites. A
ten-thousand-iteration loop recovered the PIN from a captured cookie in
milliseconds. The comment above the code said the raw PIN "never sits in the
browser cookie", which was true and beside the point.

**Shipped:** HMAC-SHA-256 keyed by `SITE_GATE_SECRET`; a warning when the
secret is absent. See [module.md](module.md).

### 3. A successful unlock re-POSTed the PIN to the landing page — verified live

`NextResponse.redirect` defaults to `307`, which preserves method and body.
The dev server log showed `POST /pl/robot 200` after the unlock — the page
rendered from a POST carrying `pin=1234` in its body — and with a locale
redirect chain the body travels once more. Reloading the landing page in a
browser prompts to resubmit the form.

**Shipped:** `303`. See [handler.md](handler.md).

### 4. A non-form body crashed the proxy — verified live

`request.formData()` throws a `TypeError` for any body that is not
`multipart/form-data` or `application/x-www-form-urlencoded`. The source did
not catch it; a `POST` with a JSON body returned a 500 page.

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

### 7. The pages-only matcher published the route list — verified live

The matcher excluded `/api` and every path with a dot. Without a cookie,
`/sitemap.xml` returned 200 and listed every URL of the unlaunched site; the
OG image and the API routes were reachable as well.

**Shipped:** the default matcher excludes build assets only, and the
pages-only shape is documented as a choice with its consequences. See
[adaptation.md](adaptation.md).

### 8. The digest was recomputed on every request

`gateToken(SITE_PIN)` ran once per request for a value that never changes.

**Shipped:** derived once per instance in `createSiteGate`.

### 9. English only, brand hardcoded, on a bilingual site

The gate page carried the site's brand as a literal and spoke English to Polish visitors whose
locale cookie was right there in the request.

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
- **`inputmode="numeric"` is gone, `type="password"` is in.** The source
  advertised the PIN's alphabet; a password field does not.

## Added

Beyond what the source had, all marked as additions above: the attempt
budget and `429`, the keyed token and `SITE_GATE_SECRET`, locale selection and
the strings table, the return-path sanitiser, `400` on a bad body,
`x-robots-tag`, `Retry-After`, the warn-level log lines, the brand env var,
and the test suite. The Redis attempt store, the lock endpoint, per-client
PINs and the bypass header in [operations.md](operations.md) are designs only.

## If you are porting the original

Fix order, most damaging first:

1. Sanitise the return path (defect 1) — one function, closes a live
   phishing vector.
2. Change the unlock redirect to `303` (defect 3).
3. Key the cookie token with a secret (defect 2) — invalidates every existing
   cookie, so schedule it.
4. Wrap `formData()` (defect 4).
5. Add the attempt budget (defect 5), reusing the host's limiter if it has one.
6. Constant-time compares (defect 6).
7. Decide the matcher on purpose (defect 7).
