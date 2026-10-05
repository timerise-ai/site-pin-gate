# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.4.0] - 2026-10-05

From applying the skill to a second host: the staging site of a multi-vendor
marketplace, with an existing proxy, nine UI locales including Arabic, and
inbound webhooks from six kinds of provider. `provenance.md` records it as
source B.

### Added

- `resolveLocale` and `localeDir` on `createSiteGate`. A host with its own
  locale negotiation hands it to the gate, which `pickLocale` could not stand
  in for: `zh-CN` never matches `zh-Hans` by base language. The gate page takes
  a `dir`, so a right-to-left locale lays out right to left, and a resolver
  result missing from the strings table renders the default locale. Two
  handler tests; the suite is 40 tests.
- The bypass-list variant in `adaptation.md`, for a host whose proxy already
  runs on nearly every path for other jobs: keep its matcher and skip the gate
  in code for webhook, cron and feed prefixes, tested at its edges.
- A machine-caller inventory in the host probe: webhooks, cron trigger URLs,
  health probes and polled feeds, and why browser redirects stay gated.
- In `operations.md`: why platform password protection refuses webhooks and
  what that costs, which environment a staging domain is really served from, a
  webhook path in the smoke checks, unlocking before a browser test run, and
  what an app built from an older copy of the skill has to port.
- In `testing.md`: `// @vitest-environment node` for a host whose runner
  defaults to `jsdom`.

### Changed

- The gate page sets `translate="no"` on `<html>` and `dir="ltr"` on the PIN
  field.
- The matcher fact in `SKILL.md`, and non-negotiable 6 in the README and in
  `adaptation.md`, name the bypass list as the boundary beside a shared matcher.
- The fix order in `provenance.md` starts by reading the changelog since the
  version a gate was built from: source B was built from 0.3.1 and carried the
  dot-segment redirect fixed in 0.3.6 until it was ported.

## [0.3.7] - 2026-09-27

Fix release, from scoring the prompt-1 agent eval runs against 0.3.6.

### Fixed

- `operations.md` told agents to list "both variables" in `.env.example`, so a
  generated app could leave `SITE_GATE_BRAND` out. It now names all three, and
  so does its checklist.

### Changed

- The invocation contract says what happens with no PIN and nobody to ask: the
  skill writes no PIN anywhere, not even a development one in `.env.local`, and
  smoke-tests with a throwaway value in the command's environment. `SKILL.md`,
  the README and `operations.md` agree.

## [0.3.6] - 2026-09-27

Security fix release, from scoring the prompt-1 agent eval runs against 0.3.5.

### Security

- `safeReturnPath` let an off-origin return path through. `/a/..//evil.example`
  starts with one slash and resolves to the request origin, but its normalised
  pathname is `//evil.example`, which the unlock redirect resolved to another
  host. A phishing link could carry anyone who knows the PIN off-site. The
  sanitiser now resolves its result once more and collapses anything that
  leaves the origin. Update the template in any app built from 0.3.5 or earlier.

### Added

- Two dot-segment cases in the `safeReturnPath` suite and one in the handler's
  off-origin test; the suite is 38 tests.

### Changed

- The return-path hard rule in `SKILL.md` and the non-negotiable in the README
  name the dot-segment case. `provenance.md` records it under ledger entry 1.
- `handler.md` says the wiring is the whole of `proxy.ts`: no lazy
  initialisation, no `503` when `SITE_GATE_SECRET` is missing, and no cookie
  format pre-check.

## [0.3.5] - 2026-09-27

Fix release, from scoring the prompt-1 agent eval runs against 0.3.4.

### Changed

- The quick start tells an agent to copy the default matcher verbatim; only a
  path the task names as public changes it. `adaptation.md` says the same, and
  says that dropping an exclusion is an unreviewed matcher shape.
- The quick start and `testing.md` say to install `vitest` with
  `npm i -D vitest` when the host has no runner: the package registry is not an
  external service. The suite runs unmodified, never converted to `node:assert`
  or run through a hand-rolled runner.
- The quick start and `operations.md` make both variables part of the handover
  when the agent cannot set them: an unset `SITE_PIN` is an open site, and
  `SITE_GATE_SECRET` belongs wherever `SITE_PIN` is.

## [0.3.4] - 2026-09-27

Fix release, from reading the prompt-1 agent eval runs against 0.3.3.

### Changed

- The matcher hard rule in `SKILL.md` and the matcher non-negotiable in the
  README now say to keep `_next/static` outside the matcher. Gating build assets
  hides nothing from a crawler, which never holds the cookie and sees only the
  `401`. `adaptation.md` says outright that `/:path*` is not a third matcher option.

### Fixed

- Both `303` redirects from the unlock path, the non-POST one and the one that
  sets the cookie, now send `cache-control: no-store`, so a shared cache cannot
  replay the redirect that carries the cookie. The behaviour contract, the
  handler, its checklist and the handler suite agree; the suite is still 36 tests.
  `provenance.md` lists it under *Added*.

## [0.3.3] - 2026-09-27

Fix release, from reading the prompt-1 agent eval runs against 0.3.2.

### Fixed

- The in-memory attempt store is now bounded under a flood of live keys. Pruning
  only dropped expired keys, so distinct client keys arriving inside one window
  grew the map past `maxKeys` and re-scanned it on every hit. At the cap it now
  evicts the oldest key, and it prunes only when a new key arrives. The module
  prose no longer claims a bound the code did not keep.
- The agent eval workflow caller passes the dispatched prompt number as a number,
  so a manual run is no longer refused before a job starts.

### Added

- A test that evicts the oldest key when every key is live; the suite is 36 tests.
- `operations.md` says to un-ignore `.env.example`, which the `.env*` rule in
  current `create-next-app` catches, and gains an *Indexing* section on why the
  default matcher already keeps a gated site out of search indexes and the matcher
  should not be widened to build assets for that.
- `testing.md` says a host with no runner gets `vitest` as a dev dependency, rather
  than a rewritten import or a hand-rolled `expect` shim.

## [0.3.2] - 2026-09-27

Documentation release. The skill content is unchanged from 0.3.1.

### Changed

- The README file table and `CLAUDE.md` list `evals/`, the prompts an operator
  types after installing and one file per agent eval run, as the skills standard
  now asks. `CLAUDE.md` adds that evals are committed as `chore(evals)` and never
  bump the version, and describes the agent eval workflow caller.
- README paragraphs that broke mid-sentence are rewrapped to 110 columns, with the
  wording unchanged.

## [0.3.1] - 2026-09-21

Wording release. The skill content is unchanged from 0.3.0.

### Added

- `SKILL.md` closes with a line linking the
  [Timerise Skills](https://github.com/timerise-ai/skills) index, so an agent that
  has the skill loaded can find the sibling skills for neighbouring modules without
  leaving the entry point.

### Changed

- `CLAUDE.md` records the closing line in the `SKILL.md` layout, and the line budget
  it states holds that line aside.

## [0.3.0] - 2026-09-05

### Added
- An uninstall argument. `/site-pin-gate uninstall` removes everything the skill
  put in the host app and nothing else: the wiring in `proxy.ts` or
  `middleware.ts`, the `lib/site-gate` directory with its tests and optional
  files, and `SITE_PIN`, `SITE_GATE_SECRET` and `SITE_GATE_BRAND` from
  `.env.local`, `.env.example` and every environment in the platform's store.
- `uninstall` as a reserved word, never a PIN, so it cannot collide with the
  single-bare-token PIN argument added in `0.2.0`.
- The procedure that argument follows, in *Uninstalling* in
  `references/operations.md`, next to the kill switch it complements. Removing
  the code opens the site everywhere, so an uninstall is treated as a launch: it
  states what will be deleted and confirms before the first removal, restores the
  host's own matcher from git history, and points back to unsetting `SITE_PIN`
  when the gate may be needed again.

### Changed
- The invocation contract still moves as one, so the reserved word is stated in
  all three of its places: the `## Invocation` table in `SKILL.md`, the
  activation paragraph in `README.md`, and `references/operations.md`.

No template, test or runtime behaviour changed: the gate this skill generates is
identical to `0.2.0`, so `references/provenance.md` gains no entry.

## [0.2.0] - 2026-09-03

### Added
- A PIN supplied at invocation. `/site-pin-gate 1111` arms the gate with that PIN
  instead of asking for one: a single bare token is read as the PIN, while more
  than one token stays a task description, so plain invocations are unchanged.
  The PIN is named back before use, so a single word meant as a topic is not
  silently armed as one.
- The rules that argument carries, read off the existing templates: 1 to 128
  characters after trimming, a reported strength estimate rather than a refusal
  for a short PIN, and the PIN written to `.env.local` and nowhere else: never a
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
