# site-pin-gate

[![Agent Skills](https://img.shields.io/badge/Agent_Skills-open_format-059669)](https://agentskills.io)
[![skills.sh](https://img.shields.io/badge/skills.sh-npx_skills_add-059669)](https://www.skills.sh)
[![Claude Code](https://img.shields.io/badge/Claude_Code-compatible-059669)](https://docs.claude.com/en/docs/claude-code/skills)
[![Codex CLI](https://img.shields.io/badge/Codex_CLI-compatible-059669)](https://developers.openai.com/codex/skills)
[![Gemini CLI](https://img.shields.io/badge/Gemini_CLI-compatible-059669)](https://github.com/google-gemini/gemini-cli/blob/main/docs/cli/skills.md)

An [Agent Skill](https://agentskills.io) that teaches an agent to put a shared-PIN gate in front of an entire
**Next.js App Router** site from the proxy or middleware layer: one env var arms it, a self-contained unlock
page sets an HMAC-derived `HttpOnly` cookie, and the app behind it stays untouched. Unset the variable and the
gate is gone.

**Everything dangerous in a gate lives in the two places it touches the outside world: the return path it
redirects to, and the cookie it hands out.** The rest is a form and a cookie check. Get those two right and
one
shared PIN is a complete answer for the weeks before a launch, a client or investor preview, or a staging
domain: everyone who should see the site can be told the PIN, and nobody else, crawlers included, sees
anything
but a 401 page. It is not user authentication and not per-route authorization, because a shared PIN cannot be
revoked for one person.

This skill was written by the engineer who has shipped this module. The earlier implementation it was audited
against was a temporary gate on a marketing site kept private ahead of its launch. The templates hold the
properties a gate has to hold: an unlock that can only return to the site's own origin, a redirect the browser
follows as a GET so the PIN is never sent twice, a cookie that is opaque to anyone without the server's
secret,
comparisons that take the same time whatever the input, a request body that is answered rather than thrown,
and
a failed-attempt budget that answers 429. The behaviour contract and the handler suite state each one;
[`references/provenance.md`](references/provenance.md) has the record.

## Install

One command, via the [skills.sh](https://www.skills.sh) CLI, which installs the skill into every
skills-compatible agent it detects, including Claude Code, Codex CLI and Gemini CLI:

```bash
npx skills add timerise-ai/site-pin-gate
```

Name the agents instead with `-a`, for example `npx skills add timerise-ai/site-pin-gate -a claude-code -a
codex`.

### Manual install

Nothing here is Claude-specific: the skill is a plain [Agent Skills](https://agentskills.io) folder,
`SKILL.md`
plus markdown references with no file that calls a model, so cloning it into an agent's skills directory is
all
an install is. For Claude Code:

```bash
git clone https://github.com/timerise-ai/site-pin-gate.git ~/.claude/skills/site-pin-gate
```

To scope it to a single project instead, clone it into that project's `.claude/skills/` directory. For another
agent, clone into that agent's skills directory, or symlink the Claude Code copy so one `git pull` updates
every agent:

```bash
mkdir -p ~/.agents/skills
ln -s ~/.claude/skills/site-pin-gate ~/.agents/skills/site-pin-gate
```

Update the skill with `git pull` in its directory. The current release is **0.1.0**. See
[`CHANGELOG.md`](CHANGELOG.md). The [skills index](https://github.com/timerise-ai/skills) lists the other
Timerise Skills and how to install them all at once.

## Activation

The skill activates automatically when a task matches its description: hiding a site behind a PIN or password
before launch, locking a preview or staging deployment, adding a coming-soon or access-code page, or auditing
an existing gate in `middleware.ts` or `proxy.ts`. Invoke it explicitly with `/site-pin-gate` in Claude Code,
`$site-pin-gate` in Codex CLI, or from `/skills` in Gemini CLI.

A single bare argument is taken as the PIN: `/site-pin-gate 1111` arms the gate with that PIN, writes it to
`.env.local` and nowhere else, and reports what a PIN that short is worth. More than one word is read as a
task description, so `/site-pin-gate audit middleware.ts` still behaves like a plain invocation.

Each host matches a task against the description its own way, so invoke the skill explicitly on a first run
rather than assuming it fired. Only `SKILL.md` is read up front; the `references/` files load on demand, so
the
skill stays cheap in context until a topic is actually needed.

## What's inside

| File | Contents |
|---|---|
| `SKILL.md` | Entry point: architecture diagram, critical facts, hard rules, quick start, and the reference directory |
| `references/adaptation.md` | The seam contract with the host app: proxy versus middleware, the matcher, strings, styling, cookie name, a shared attempt store |
| `references/module.md` | Config, token derivation, constant-time compare, the return-path sanitiser, the attempt store |
| `references/handler.md` | The behaviour contract, the gate page, the request handler, the proxy wiring |
| `references/operations.md` | Env vars per environment, a PIN given at invocation, smoke checks, rotation, kill switch, what stays public, extensions |
| `references/testing.md` | The two test files, 35 tests, and how to run them under vitest or bun |
| `references/provenance.md` | The engineering ledger: what the audit of the earlier implementation changed and how the templates verify it, what was kept on purpose, and what is new in the skill |

The seam is the table at the top of `references/adaptation.md`, and it is short because the gate keeps no
state
beyond one cookie and one counter: there is no domain vocabulary to rename, no tenant scope and no data
access.
The attempt store is the one pluggable dependency, in-memory by default with a shared-store adapter sketched
for a hard global cap. The matcher, cookie name, unlock path, strings, palette, locale detection and logger
are
the host app's, and no template adds a dependency: they import only `next/server` and Web Crypto.

## The six non-negotiables

These travel with the module and are never optional. Each is stated as a hard rule in `SKILL.md` and covered
by
the suite in `references/testing.md`:

1. **The return path is resolved against the request origin**, never checked with `startsWith('/')`. A path
   that begins with a slash can still leave the site, so the sanitiser resolves the value and compares
   origins.
   Three off-origin cases are in the suite.
2. **The unlock redirect is a 303.** A 307 tells the browser to repeat the request with its method and
   body, so the PIN would be posted again to the landing page, and once more through any locale redirect.
   The status is asserted on the success path.
3. **The cookie is an HMAC keyed by a server-side secret**, not the PIN and not an unkeyed hash of it. A
   cookie an operator can read is a cookie an operator's laptop can leak. A token derived without the
   secret is rejected in the suite.
4. **Both comparisons are constant-time** over equal-length digests, so neither the PIN check nor the cookie
   check leaks its answer through how long it took.
5. **A body that is not a form is answered, never thrown.** `request.formData()` rejects anything that is not
   form-encoded, and an unhandled rejection in the proxy is a stack trace instead of a rejected attempt. The
   400 path is in the suite.
6. **The matcher is chosen on purpose, and the unlock path sits outside every locale prefix and app route.**
   Whatever the matcher excludes is public, including the sitemap and OG images unless you say otherwise; a
   colliding unlock path would never render, because the gate answers it before routing.

Everything else is the host app's: cookie name, unlock path, strings, palette, locale detection, attempt
store.

## Not this

| Not this | Use instead |
|---|---|
| Real users with real accounts | The host's auth: Clerk, NextAuth, Supabase Auth. A shared PIN cannot be revoked for one person |
| Per-route or per-tenant authorization | The host's authorization layer; this gate is all-or-nothing by path matcher |
| Locking preview deployments for team members on Vercel | Vercel Deployment Protection, which does it with no code. The comparison is in `references/operations.md` |
| Protecting an API consumed by machines | An API key; a cookie and an HTML form are the wrong shape |

## Contributing

Issues and pull requests are welcome here. Pure markdown, with no build step, but the code blocks are checked:
every TypeScript block names its destination on the first line, and the module, handler and test blocks are
written to compile as one project under `strict` and `noUncheckedIndexedAccess` and to run under `bun test`,
35
tests. Claims in this skill are meant to be verifiable: if you change a factual claim, say how you verified
it,
whether against the library, the HTTP specification, the URL parser, or a reproduction.

Adding, removing or renaming a file in `references/` means updating the quick start and the reference
directory
table in `SKILL.md`, the file table above, and any relative cross-links. Every odd-looking part of the
templates is there for a reason, and `references/provenance.md` is the ledger that must stay truthful: read it
before simplifying anything, and add an entry for anything you change. Commits follow Conventional Commits and
releases follow [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) in the index;
`CLAUDE.md` carries the full editing conventions.

## Part of the Timerise Skills

This is one of the [Timerise Skills](https://github.com/timerise-ai/skills): modules for **Next.js App
Router** apps written by our own senior engineers from the modules they have shipped, not synthetic, each
published as its own repository and indexed there. They share one layout, so an agent that has read one knows
how to read the next: a `SKILL.md` entry point, `references/` loaded on demand, and a seam contract carrying
the module's non-negotiables.

## Author

Built and maintained by [Timerise](https://timerise.ai).

## License

MIT. See [LICENSE](LICENSE).
