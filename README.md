# site-pin-gate

An [Agent Skill](https://agentskills.io) that teaches an agent to put a shared-PIN gate in front of an
entire **Next.js App Router** site from the proxy or middleware layer. One env var arms it, a
self-contained unlock page sets an HMAC-derived `HttpOnly` cookie, and the app behind it stays untouched.
Unset the variable and the gate is gone.

It is for the weeks before a launch, a client or investor preview, a staging domain: everyone who should
see the site can be told one PIN, and nobody else — including crawlers — sees anything but a 401 page.
It is not user authentication and not per-route authorization; a shared PIN cannot be revoked for one
person.

The skill is extracted from a production module, and the extraction audit found that everything dangerous
lived in the two places a gate touches the outside world: the return path it redirects to, and the cookie
it hands out. The source accepted `//evil.example` as a same-site path and redirected there after a
correct PIN; derived the cookie as an unkeyed hash from which a numeric PIN was recovered in milliseconds;
answered a successful unlock with a `307` that re-POSTed the PIN to the landing page; and returned a 500
for any POST that was not a form. Each was reproduced live before being fixed in the templates, and each
fix has a test.

## Install

Clone into your skills directory:

```bash
git clone https://github.com/timerise-ai/site-pin-gate.git ~/.claude/skills/site-pin-gate
```

To scope it to a single project instead, clone it into that project's `.claude/skills/` directory. Update
it with `git pull` in its directory. Current release: **0.1.0** — see [`CHANGELOG.md`](CHANGELOG.md).

### One command, via skills.sh

```bash
npx skills add timerise-ai/site-pin-gate                           # every agent it detects
npx skills add timerise-ai/site-pin-gate -a claude-code -a codex   # or just the ones you name
```

### Codex CLI, Gemini CLI and other agents

Nothing here is Claude-specific. `SKILL.md` declares only `name` and `description`; everything below it
is plain markdown about Next.js, cookies and HTTP. Codex CLI and Gemini CLI read the same layout and
discover skills in `~/.agents/skills`:

```bash
mkdir -p ~/.agents/skills
git clone https://github.com/timerise-ai/site-pin-gate.git ~/.agents/skills/site-pin-gate
```

If the skill is already cloned for Claude Code, symlink it rather than cloning it twice:

```bash
ln -s ~/.claude/skills/site-pin-gate ~/.agents/skills/site-pin-gate
```

The skill activates when a task matches its description — hiding a site behind a PIN or password before
launch, locking a preview or staging deployment, or auditing an existing gate in `middleware.ts` or
`proxy.ts`. In Claude Code you can also invoke it explicitly with `/site-pin-gate`.

## What's inside

| File | Contents |
|---|---|
| `SKILL.md` | Entry point: architecture, critical facts, hard rules, and the reference directory |
| `references/adaptation.md` | The seam contract: proxy vs middleware, the matcher, strings, styling, cookie name, a shared attempt store |
| `references/module.md` | Config, token derivation, constant-time compare, the return-path sanitiser, the attempt store |
| `references/handler.md` | The gate page, the request handler, the behaviour contract, the proxy wiring |
| `references/operations.md` | Env vars per environment, smoke checks, rotation, kill switch, what stays public, extensions |
| `references/testing.md` | The two test files and how to run them under vitest or bun |
| `references/provenance.md` | The twelve findings from the original module and how the templates handle each |

References are loaded on demand — the agent reads only the ones a task needs.

## The five non-negotiables

1. **The return path is resolved against the origin**, never checked with `startsWith('/')`.
2. **The unlock redirect is a 303**, so the browser never re-POSTs the PIN.
3. **The cookie is an HMAC keyed by a server-side secret**, not a hash of the PIN.
4. **Both comparisons are constant-time** over equal-length digests.
5. **The matcher is chosen on purpose.** What it excludes is public.

Everything else — cookie name, unlock path, strings, palette, attempt store — is the host app's.

## Contributing

Pure markdown, but the code blocks are checked: every ` ```ts ` block starting with `// file: <path>` is
extracted into a scratch project and type-checked under `strict` and `noUncheckedIndexedAccess`, and the
`*.test.ts` blocks are run. Adding, removing or renaming a file in `references/` means updating the
reference directory in `SKILL.md`, the table above, and any relative cross-links. Complexity in the
templates encodes a documented defect: check `references/provenance.md` before simplifying anything.
See `CLAUDE.md` for the editing conventions.

## Part of the Timerise skills

One of the [Timerise skills](https://github.com/timerise-ai/skills) — production-extracted modules for
Next.js App Router apps, each its own repository, sharing one layout: a `SKILL.md` entry point,
`references/` loaded on demand, and a `provenance.md` that says what was found in the source and why the
templates differ.

## License

MIT — see [`LICENSE`](LICENSE).
