---
prompts:
  - prompt: "Hide this whole site behind a PIN until launch. One environment variable turns it on, and search engines must not index anything while it is on."
    stack: No data store
  - prompt: "Lock our staging deployment for a client review with a shared PIN, and slow down anyone who keeps guessing wrong."
    stack: No data store
  - prompt: "Audit the password gate in our middleware.ts."
---

# Prompts

What an operator types after installing this skill, in their own words. An agent eval installs the skill
into an empty Next.js app, gives the agent one of these prompts and no further help, then type-checks, builds
and tests the result; the first prompt runs before every release. The results are the other files in this
folder. Section 10 of [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) says how a
run is made. The prompts and the newest runs are on
[the skill's page](https://timerise.ai/skills/site-pin-gate) on timerise.ai.
