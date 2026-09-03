# Operations

The gate has no admin surface. Everything an operator does is an env var and a
`curl`.

## Arming it, per environment

The failure to design against: **the PIN is set in one environment and not
another, and the unlocked environment is the one that matters.** An unset PIN
is a fully open site with no warning anywhere.

| Environment | `SITE_PIN` | `SITE_GATE_SECRET` | Result |
|---|---|---|---|
| Production, pre-launch | set | set | Locked |
| Production, launched | **unset** | either | Open, this is how you launch |
| Preview / staging | set | set | Locked |
| Local development | set in `.env.local` when you need to see the gate; otherwise unset | optional | Cookie is sent without `Secure` because `NODE_ENV` is not `production` |

On Vercel:

```bash
vercel env add SITE_PIN production      # paste the PIN when prompted
vercel env add SITE_PIN preview
vercel env add SITE_GATE_SECRET production
vercel env add SITE_GATE_SECRET preview
openssl rand -hex 32                    # a fine SITE_GATE_SECRET
```

Env changes take effect on the **next deployment**. Redeploy after setting or
removing either variable; the running functions keep the old values.

Keep `.env.example` honest: list both variables with empty values and a
comment that empty means off.

## Smoke checks

Run against every environment after every deploy that touches the gate:

```bash
H=https://preview.example.com
curl -sI $H/                 | head -1        # 401 expected while locked
curl -sI $H/sitemap.xml      | head -1        # 401 with the default matcher
curl -sI $H/api/health       | head -1        # whatever you decided for it
curl -sI -X POST $H/__unlock -H 'content-type: application/x-www-form-urlencoded' \
     --data 'pin=WRONG&next=/' | head -1      # 401, not 500
```

And once with the real PIN, checking the redirect is a `303` to your own
host:

```bash
curl -sI -X POST $H/__unlock -H 'content-type: application/x-www-form-urlencoded' \
     --data "pin=$SITE_PIN&next=//evil.example" | grep -i '^HTTP\|^location'
# HTTP/2 303
# location: https://preview.example.com/
```

## Rotating the PIN

Change `SITE_PIN`, redeploy. Every issued cookie was derived from the old PIN,
so every browser is locked out at once and asked again. There is no session
table to clear. The same happens if you change `SITE_GATE_SECRET` or bump the
`site-gate:v1:` prefix in `deriveGateToken`.

## Kill switch

Unset `SITE_PIN`, redeploy. The gate becomes a no-op. To remove the code as
well, delete the `lib/site-gate` directory and the four lines in the proxy;
nothing else references it.

## What stays public

With the default matcher, only `_next/static`, `_next/image` and
`favicon.ico`. Those are hashed build assets; they contain client-component
code and any string in it, which the gate cannot protect. Do not put secrets
in client code.

With the pages-only matcher (see [adaptation.md](adaptation.md)), also:
`/api/*`, `sitemap.xml`, `robots.txt`, every generated OG image, and every
file under `public/`. The earlier implementation shipped this way, which
meant its sitemap listed every route of an unlaunched site to anyone who
asked. Choose
it only when a third party must reach those paths without the cookie, and
prefer excluding the specific paths.

## What the logs show

The handler writes one `warn` line per failed attempt and one when a client's
budget is exhausted, each with the client key. It writes one `warn` per
instance when `SITE_GATE_SECRET` is unset. Successful unlocks are not logged;
add a line if you want an audit trail of who got in; there is no identity to
record beyond the client key.

## Compared with Vercel Deployment Protection

| | This gate | Vercel Authentication | Vercel Password Protection |
|---|---|---|---|
| Who can enter | Anyone with the PIN | Members of the Vercel team | Anyone with the password |
| Covers | Whatever the matcher covers, in any environment | Preview deployments (production optional) | Preview and production |
| Cost | None | Included on all plans | Paid add-on on Pro; included on Enterprise |
| Localised page | Yes, your strings | No | No |
| Removal | Unset an env var | A dashboard toggle | A dashboard toggle |

Check current plan details before relying on the cost row. If the site is on
Vercel, the audience is the team, and only previews need hiding, use Vercel
Authentication and skip this module.

## Extensions

Designs, not shipped code. Each is small; none was needed by the earlier
implementation.

**A lock endpoint.** A `GET /__lock` that clears the cookie and redirects to
`/`, for demonstrating the gate or handing a shared laptop back. Ten lines in
the handler: match the path, `response.cookies.delete(config.cookieName)`,
redirect `303`.

**Per-client PINs.** Accept a comma-separated `SITE_PIN`, derive one expected
token per PIN, and accept a cookie that matches any of them. Log which index
unlocked. Revoking one client is then removing one PIN. Keep the compare
constant-time across the whole list (compare against every token, OR the
results) or the count of PINs leaks.

**Shared attempt store.** The Redis sketch in [adaptation.md](adaptation.md).
Worth it when the platform runs many instances and the PIN is short; not
worth it when the PIN is long.

**Bypass header for uptime monitors.** A `SITE_GATE_BYPASS_TOKEN` compared,
constant-time, against an `x-site-gate-bypass` header. Simpler than
excluding paths, but a second secret to manage.

## Checklist

- [ ] `SITE_PIN` and `SITE_GATE_SECRET` set in every environment that should be
      locked, and **unset** where it should be open
- [ ] Redeployed after every env change
- [ ] Smoke checks pass on every environment
- [ ] `.env.example` lists both variables
- [ ] The team knows that changing the PIN logs everyone out
