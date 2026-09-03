# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.0] - 2026-09-03

### Added
- A PIN supplied at invocation. `/site-pin-gate 1111` arms the gate with that PIN
  instead of asking for one: a single bare token is read as the PIN, while more
  than one token stays a task description, so plain invocations are unchanged.
  The PIN is named back before use, so a single word meant as a topic is not
  silently armed as one.
- The rules that argument carries, read off the existing templates: 1 to 128
  characters after trimming, a reported strength estimate rather than a refusal
  for a short PIN, and the PIN written to `.env.local` and nowhere else — never a
  template, a test, `.env.example` or a commit, and never `SITE_GATE_SECRET`,
  which stays generated.
- The invocation contract now lives in three places that move as one: the
  `## Invocation` table in `SKILL.md`, the activation paragraph in `README.md`,
  and *A PIN supplied at invocation* in `references/operations.md`.

No template, test or runtime behaviour changed: the gate this skill generates is
identical to `0.1.0`, so `references/provenance.md` gains no entry.

## [0.1.0] - 2026-09-03

Initial release of the `site-pin-gate` skill: a shared-PIN gate for a whole Next.js
App Router site, armed by one env var from `proxy.ts` or `middleware.ts`.

### Added
- `SKILL.md` entry point: the architecture diagram, five critical facts, six hard
  rules, the quick-start order, and the reference directory table mapping trigger
  keywords to `references/`.
- `references/adaptation.md`: the seam contract with the host app, `proxy.ts` versus
  `middleware.ts`, choosing the matcher, strings per locale, styling, cookie name and
  unlock path, and a shared attempt store.
- `references/module.md`: configuration from the environment, the secret-keyed token
  derivation, the constant-time compare, the return-path sanitiser and the attempt
  store.
- `references/handler.md`: the behaviour contract, the self-contained gate page, the
  request handler and the proxy wiring.
- `references/operations.md`: env vars per environment, smoke checks, PIN rotation,
  the kill switch, what stays public, and extensions offered as designs.
- `references/testing.md`: `core.test.ts` and `handler.test.ts`, 35 tests, run under
  `vitest` or `bun test`.
- `references/provenance.md`: the engineering ledger, twelve entries on what the audit
  of the earlier implementation changed and how the templates verify it, what was kept
  deliberately, and what is new in the skill.
- Templates for `lib/site-gate/{config,core,attempts,page,handler}.ts` and a `proxy.ts`
  wiring, written to compile under `strict` and `noUncheckedIndexedAccess`, importing
  only `next/server` and Web Crypto.
- The properties the templates hold: a return path resolved against the request origin,
  a `303` unlock redirect, an HMAC cookie token keyed by `SITE_GATE_SECRET`,
  constant-time comparisons, a `400` on a body that is not a form, a per-client attempt
  budget answering `429` with `Retry-After`, `x-robots-tag: noindex, nofollow`, locale
  selection with a strings table, and the brand read from the environment.
- `README.md`: install, activation, the file table, the six non-negotiables, the
  *Not this* table and the contributing conventions.
- `CLAUDE.md`: editing conventions for this repository.
- `LICENSE`: MIT.
