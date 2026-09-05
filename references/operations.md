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

## A PIN supplied at invocation

`/site-pin-gate 1111` supplies `SITE_PIN` directly, and it lands in exactly one
file. Confirm that file is git-ignored first; `create-next-app` ignores
`.env*.local` already.

```bash
grep -n '^SITE_PIN=' .env.local 2>/dev/null   # already set? edit, do not append
printf 'SITE_PIN=%s\n' '1111' >> .env.local
printf 'SITE_GATE_SECRET=%s\n' "$(openssl rand -hex 32)" >> .env.local
```

Single-quote the PIN in the shell so `$` and spaces survive, and quote it in
the file too if it contains a space or a `#`. Both the env value and the
submitted one are trimmed, so a PIN with leading or trailing whitespace can
never be typed back.

Nothing else changes: `.env.example` still lists both names with empty values,
the deployed environments still take theirs from the platform, and no template,
test or commit carries the value.

A PIN passed as an argument has been through a shell history and an agent
transcript before it reached the file. That is fine for local work and for a
preview nobody has been given yet. Before it guards anything that matters,
rotate it: one variable, one deploy, and every cookie issued under the old
PIN dies with it.

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

Unset `SITE_PIN`, redeploy. The gate becomes a no-op and the code stays, ready
to lock the site again. This is how you launch. To take the code out as well,
see *Uninstalling* below.

## Uninstalling

`/site-pin-gate uninstall` removes everything the skill put in the host and
nothing else. `uninstall` is a reserved word, never a PIN. Prefer the kill
switch while the gate may be needed again; uninstall once the launch is
behind you and the module should not stay in the codebase.

**Uninstalling is a launch.** The moment the code is gone, every environment
is public, the platform's env store included. Say what will be deleted and
that the site opens, and get a yes before the first removal.

Find the footprint first. The skill added nothing outside what this grep
finds and the wiring file:

```bash
grep -rn 'site-gate\|site_access\|__unlock\|SITE_PIN\|SITE_GATE_' \
  --exclude-dir=node_modules --exclude-dir=.next --exclude-dir=.git .
git log --oneline -- lib/site-gate proxy.ts middleware.ts src/proxy.ts src/middleware.ts | head
```

If the host renamed the cookie or the unlock path (see
[adaptation.md](adaptation.md)), grep for those names too.

Then, in this order:

1. **The wiring.** In `proxy.ts` or `middleware.ts`: the `createSiteGate`
   import, the module-scope `siteGate` instance and the two-line early return.
   If the skill created the file and nothing but the gate lives in it, delete
   the file, `config.matcher` included. If the host had its own proxy, remove
   only those lines and restore its matcher to what it was before the gate,
   from git history: the gate's default matcher is wider than most apps need,
   and left behind it routes every request through the proxy for nothing.
2. **The module.** Delete the `lib/site-gate` directory (or `src/lib/site-gate`).
   That takes the two test files and the optional `strings.ts`, `instance.ts`
   and `redis-attempts.ts` with it; nothing outside the directory imports it
   except the wiring.
3. **The env.** Remove `SITE_PIN`, `SITE_GATE_SECRET` and `SITE_GATE_BRAND`
   from `.env.local` and `.env.example`, then from every environment in the
   platform's store:

   ```bash
   vercel env rm SITE_PIN production
   vercel env rm SITE_PIN preview
   vercel env rm SITE_GATE_SECRET production
   vercel env rm SITE_GATE_SECRET preview
   vercel env ls | grep SITE_   # nothing left, including SITE_GATE_BRAND
   ```

   The store is the step to be careful about. A `SITE_PIN` left there with no
   code reading it is a lock an operator will believe is on.
4. **Anything else the skill touched.** A test-runner include pattern or
   script added for `lib/site-gate`, and any line added to the host's README,
   `CLAUDE.md` or `.env.example` comments. Rare: the templates run in the
   host's runner as they are.
5. **Redeploy**, then verify:

   ```bash
   grep -rn 'site-gate\|site_access\|SITE_PIN\|SITE_GATE_' \
     --exclude-dir=node_modules --exclude-dir=.next --exclude-dir=.git .   # nothing
   npx tsc --noEmit
   curl -sI https://preview.example.com/ | head -1   # 200: open, on purpose
   ```

What is not removed, and why it does not matter:

- **The cookie in visitors' browsers.** `site_access` lives up to thirty days
  and nothing reads it now, so it is inert. Only if the host reuses the name
  for something else should it be expired first, with one deploy that sets
  the cookie to an empty value and `Max-Age=0`.
- **Shared attempt-store keys.** If the Redis adapter was wired, its keys
  expire on their own within one window (15 minutes by default).
- **Git history.** Nothing to scrub: the PIN was only ever in `.env.local`
  and the platform store, never in a commit.

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
- [ ] A PIN given at invocation is in `.env.local` only, and is rotated
      before it guards production
- [ ] The team knows that changing the PIN logs everyone out
- [ ] After `uninstall`, the grep finds nothing, no `SITE_` variable is left in
      the platform store, and every environment answers `200` on purpose
