# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A **skill package** — markdown only, no application, no build step. It teaches an agent how to put a shared-PIN gate in front of a whole Next.js App Router site from `proxy.ts` (Next 16) or `middleware.ts` (Next 13–15): one env var, one HMAC-derived cookie, a self-contained unlock page, an attempt budget.

The skill was extracted from a production module; `references/provenance.md` records the twelve findings from it and how the templates handle each. That file is the rationale layer — read it before "simplifying" anything.

## Commands

There is no build, lint, dev server or test runner **in this repo** — there is no `package.json` and nothing here imports anything. The only verification is compiling and running the code blocks, which requires a scratch project because they import `next/server`:

```bash
S=$(mktemp -d) && cd "$S" && npm init -y >/dev/null
npm i -D typescript next vitest @types/node @types/react >/dev/null
mkdir -p lib/site-gate
# extract every ts block whose first line is `// file: <path>` to that path,
# from references/{module,handler,testing}.md
cat > tsconfig.json <<'JSON'
{"compilerOptions":{"strict":true,"noUncheckedIndexedAccess":true,"noEmit":true,"skipLibCheck":true,
 "target":"ES2022","module":"ESNext","moduleResolution":"bundler","lib":["ES2022","DOM"],
 "types":["node"],"paths":{"@/*":["./*"]}}}
JSON
npx tsc --noEmit          # strict + noUncheckedIndexedAccess, must be clean
bun test lib/site-gate    # 35 tests; bun rewrites the `vitest` import to its own runner
```

Run a single test file with `bun test lib/site-gate/core.test.ts`, a single case with `bun test -t 'never redirects off-origin'`.

Caveats when extracting:

- `skipLibCheck` is not optional: without it `next`'s own `.d.ts` files fail the check, which says nothing about these templates. `vitest` and `@types/react` are installed for types only. Do not add `baseUrl` — TypeScript 6 removed it; the bare `paths` above is what resolves `@/*`.
- The five module files (`config.ts`, `core.ts`, `attempts.ts`, `page.ts`, `handler.ts`), the two test files and `proxy.ts` form one coherent project. `proxy.ts` needs the `@/*` path alias above.
- Three blocks in `adaptation.md` are **variants and sketches**, not part of that project: `proxy-with-i18n.ts` (needs `next-intl` and a host `@/i18n/routing`), `matcher-with-allowlist.ts` (an alternative `config` export), `redis-attempts.ts` (compiles against `./attempts` only). Type-check them separately or not at all; never let them overwrite `proxy.ts`.
- Re-run the check after editing **any** block. A block is the source of truth for a file that other blocks import.

The host-side commands documented for a *consumer* of this skill — the probe in `adaptation.md`, the `curl` smoke checks and `vercel env add` calls in `operations.md` — are content, not this repo's workflow. Keep them runnable, but they are not run here.

## Structure

- `SKILL.md` — entry point. The frontmatter `description` is the trigger surface. The body carries the architecture, critical facts, hard rules, and the **reference directory table** mapping trigger keywords to `references/`.
- `references/*.md` — one topic per file, loaded on demand. `adaptation.md` (seam contract, matcher, strings) and `module.md` (config + pure helpers + store) are the design entry points; `handler.md` carries the page, the handler and the wiring; `operations.md` the env matrix and smoke checks; `testing.md` the suites; `provenance.md` the audit.

## The architecture the templates describe

Spread across `module.md` and `handler.md`; worth holding in mind before editing either.

```
proxy.ts ──► siteGate(request) ──► NextResponse to send, or null to continue
             (handler.ts, one instance per server: expected token derived once,
              attempt store lives across requests)
                │
                ├─ config.ts    readSiteGateConfig(env, overrides) → SiteGateConfig
                ├─ core.ts      deriveGateToken · constantTimeEqual · safeReturnPath
                │               escapeHtml · clientKey · pickLocale   (no framework import)
                ├─ attempts.ts  AttemptStore · createMemoryAttemptStore (seam: Redis)
                └─ page.ts      renderGatePage · GateStrings · GATE_STRINGS_EN
```

`handler.ts` is the only file that knows `NextRequest`; everything it *decides* is a call into `core.ts` or `attempts.ts`, which is why those are unit-testable with no request object. The response for every request shape is the **behaviour contract table** at the top of `handler.md` — it is the spec, `handler.test.ts` walks its rows, and a change to one is a change to both.

## Editing conventions

- **Code blocks are compiled and run.** Every ` ```ts ` block whose first line is `// file: <path>` is extracted into a scratch project (recipe above) and type-checked under `strict` and `noUncheckedIndexedAccess`; the `*.test.ts` blocks run under `bun test`. Keep that first line, keep imports complete, and re-run after editing any block.
- **Identifiers are shared across files.** `SiteGateConfig`, `readSiteGateConfig`, `SITE_GATE_DEFAULTS`, `deriveGateToken`, `constantTimeEqual`, `safeReturnPath`, `escapeHtml`, `clientKey`, `pickLocale`, `AttemptStore`, `createMemoryAttemptStore`, `GateStrings`, `GATE_STRINGS_EN`, `renderGatePage`, `createSiteGate`, and the env names `SITE_PIN`, `SITE_GATE_SECRET`, `SITE_GATE_BRAND` appear in several references. Rename in all of them or none.
- **Keep the reference directory in sync** with `references/` — the table in `SKILL.md` and the file table in `README.md` both list every reference. Links are relative: `[x.md](references/x.md)` from SKILL.md, `[x.md](x.md)` between references.
- **Do not remove the odd-looking parts.** The `303`, the origin comparison in `safeReturnPath`, HMAC instead of a hash, the digest-vs-digest compare, budget charged before the PIN check, inline CSS, `type="password"`, `SameSite=Lax`, the `401` — each is a documented defect or a deliberate choice. Check `provenance.md` first.
- **Five non-negotiables** are listed in `README.md`. Never present them as optional elsewhere.
- Additions beyond the source module are marked as such in `provenance.md` ("Added"). New capability goes there, or in `operations.md` under "Extensions" as a design.

## Counted claims that must not drift

Several numbers are asserted in prose and are checkable against the files:

- **Twelve findings** — `provenance.md` has twelve numbered entries under "Fixed in the templates"; the count is repeated in `CHANGELOG.md` and at the top of this file. Adding an entry means updating all of them.
- **35 tests** — claimed in `testing.md` and `CHANGELOG.md`, and it is what `bun test` prints (26 `it` cases plus one `it.each` of nine).
- **Current release** — `README.md`'s "Current release" line must name the newest `CHANGELOG.md` heading. Releases are Keep a Changelog + SemVer; there is no version field in the `SKILL.md` frontmatter.
- **What is a design, not shipped code** — `provenance.md` ends by naming them (the Redis store, the lock endpoint, per-client PINs, the bypass header). `adaptation.md` and `operations.md` must keep saying so at each one.
