# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
