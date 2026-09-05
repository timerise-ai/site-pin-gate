# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

An [Agent Skill](https://agentskills.io) package: markdown only. There is no `package.json` here and nothing
in this repository executes. It teaches an agent how to put a shared-PIN gate in front of a whole **Next.js
App Router** site from `proxy.ts` (Next 16) or `middleware.ts` (Next 13 to 15): one env var, one HMAC-derived
cookie, a self-contained unlock page, an attempt budget.

Keep the two straight: the commands and code in `references/` describe the app the agent will generate, not
this repository. The `curl` smoke checks and `vercel env add` calls in `operations.md`, the host probe in
`adaptation.md` and the `vitest` and `bun test` invocations in `testing.md` all run in that generated app.
The one thing that is checked here is that the templates compile and their tests pass, and that check runs in
a scratch project; the recipe is under *Editing conventions* below.

The skill was written by the engineer who has shipped this module; the earlier implementation it was audited
against was a temporary gate on a marketing site kept private ahead of its launch. `references/provenance.md`
is the ledger of that audit: twelve entries on what changed and how the templates verify it, what was kept
deliberately, and what was designed here and has never run in production. That file is the rationale layer:
read it before "simplifying" anything.

## Structure

- `SKILL.md`: entry point, loaded whole on every activation, so it stays between 130 and 160 lines. The
  frontmatter `description` is the trigger surface; the body carries the architecture diagram, five **critical
  facts**, six **hard rules**, the quick-start order, and the **reference directory table** mapping trigger
  keywords to files.
- `README.md`: the human-facing front door, in the section order of the skill standard: install, activation,
  the file table, the six non-negotiables, the *Not this* table, contributing.
- `references/*.md`: one topic per file, loaded on demand. `adaptation.md` (the seam contract, the matcher,
  strings) and `module.md` (config, pure helpers, attempt store) are the design entry points; `handler.md`
  carries the behaviour contract, the page, the handler and the wiring; `operations.md` the env matrix and the
  smoke checks; `testing.md` the two suites; `provenance.md` the audit.

## Editing conventions

- **Code blocks name their destination on the first line** as a comment, for example
  `// file: lib/site-gate/core.ts`.
  That line is what makes a block extractable, so keep it and keep imports complete.
- **The code blocks are compiled and run.** The blocks in `module.md`, `handler.md` and `testing.md` form one
  project: extract each to its named path in a scratch directory, then

  ```bash
  npm i -D typescript next vitest @types/node @types/react
  npx tsc --noEmit          # strict, noUncheckedIndexedAccess, skipLibCheck, paths {"@/*": ["./*"]}
  bun test lib/site-gate    # 35 tests; bun rewrites the `vitest` import to its own runner
  ```

  `skipLibCheck` is not optional, or Next's own type declarations fail the run and say nothing about these
  templates. Do not set `baseUrl`, which TypeScript 6 removed; the bare `paths` entry resolves `@/*` for
  `proxy.ts`. Re-run after editing any block. The three blocks in `adaptation.md` are variants and sketches,
  not part of that project: `proxy-with-i18n.ts` needs `next-intl` and a host `@/i18n/routing`,
  `matcher-with-allowlist.ts` is an alternative `config` export, and `redis-attempts.ts` compiles against
  `./attempts` alone.
- **Identifiers are shared across files.** `SiteGateConfig`, `readSiteGateConfig`, `SITE_GATE_DEFAULTS`,
  `deriveGateToken`, `constantTimeEqual`, `safeReturnPath`, `escapeHtml`, `clientKey`, `pickLocale`,
  `AttemptStore`, `createMemoryAttemptStore`, `GateStrings`, `GATE_STRINGS_EN`, `renderGatePage`,
  `createSiteGate`, and the env names `SITE_PIN`, `SITE_GATE_SECRET`, `SITE_GATE_BRAND` appear in several
  references. Rename in all of them or none.
- **Keep the three tables in sync** with `references/`: the reference directory in `SKILL.md`, the quick-start
  list in `SKILL.md`, and the file table in `README.md`. Links are relative: `[x.md](references/x.md)` from
  `SKILL.md`, `[x.md](x.md)` between references.
- **The invocation contract lives in three places** and moves as one: the `## Invocation` table in
  `SKILL.md`, the activation paragraph in `README.md`, and *A PIN supplied at invocation* plus
  *Uninstalling* in `operations.md`. A PIN given as an argument is written to `.env.local` and nowhere
  else: never a template, a test, `.env.example` or a commit, and never `SITE_GATE_SECRET`, which is
  generated. `uninstall` is a reserved word, never a PIN: it removes only what the skill added (the
  wiring, `lib/site-gate`, the three env vars in every store), it is a launch, and it confirms before the
  first removal. Keep the three places agreeing on that.
- **The behaviour contract is the spec.** The table at the top of `handler.md` states the response for every
  request shape, `handler.test.ts` walks its rows, and a change to one is a change to both.
- **Do not remove the odd-looking parts.** The `303`, the origin comparison in `safeReturnPath`, HMAC instead
  of a bare hash, the digest-against-digest compare, the budget charged before the PIN is checked, the inline
  CSS, `type="password"`, `SameSite=Lax` and the `401` on both error branches: each is a ledger entry or a
  documented judgement call. Check `provenance.md` before touching one.
- **The numbers that remain are load-bearing.** 35 tests, twelve ledger entries, the attempt defaults (5 tries
  per 15 minutes), the 30-day cookie, the 128-character PIN cap. They were verified against this repository or
  are design parameters the next implementation needs. Do not restate them loosely and do not add new ones.
  Figures describing the earlier implementation's deployment do not appear anywhere.
- **Mark additions as additions.** Anything designed in the skill and never run in the earlier implementation
  belongs in the "Added" section of `provenance.md`, stated as such, or in `operations.md` under *Extensions*
  as a design. The skill's credibility is that it distinguishes the two.
- **Never present the non-negotiables as optional.** The origin-resolved return path, the `303`, the
  secret-keyed cookie, the constant-time comparisons, the answered non-form body and the deliberate matcher
  are stated as hard rules in `SKILL.md` and as non-negotiables in `README.md`; keep them that way everywhere.
- The cookie name `site_access`, the unlock path `/__unlock` and the brand default are meant to be renamed by
  the host, and `adaptation.md` carries that procedure. The env var names are the operator contract and are
  not renamed.
