# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] - 2026-09-03

### Added
- Initial extraction from a production Next.js 16 site's temporary `SITE_PIN`
  gate: `SKILL.md`, `references/{adaptation,module,handler,operations,testing,provenance}.md`.
- Templates: `lib/site-gate/{config,core,attempts,page,handler}.ts` and a
  `proxy.ts` wiring, type-checked under `strict` and `noUncheckedIndexedAccess`.
- Tests: `core.test.ts` and `handler.test.ts` (35 cases), run under `bun test`.
- Fixes for the twelve findings recorded in `references/provenance.md`, four of
  them reproduced live: an open redirect after unlock, a cookie that leaked the
  PIN, a `307` that re-POSTed the PIN, a `500` on non-form bodies.
- Additions beyond the source, marked as such: attempt budget with `429` and
  `Retry-After`, `SITE_GATE_SECRET`-keyed HMAC token, locale-aware strings,
  `x-robots-tag`, `400` on a bad body, brand from env.
