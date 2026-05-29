# FeathersJS plugin for Claude Code

[![Latest release](https://img.shields.io/github/v/release/hassan4702/feathers-plugin?sort=semver)](https://github.com/hassan4702/feathers-plugin/releases)
[![License: MIT](https://img.shields.io/github/license/hassan4702/feathers-plugin)](LICENSE)
[![Claude Code plugin](https://img.shields.io/badge/Claude%20Code-plugin-8A63D2)](https://code.claude.com/docs/en/plugins)

Teaches Claude Code to build [**FeathersJS v5 (Dove)**](https://feathersjs.com) APIs the idiomatic way — the four-file service pattern, around/before/after hooks, and TypeBox schemas with resolvers — and to review them for the security mistakes that bite Feathers apps.

> **This is a public, community-maintained plugin.** Anyone can install it with the commands below. It is **not affiliated with, maintained by, or endorsed by** the FeathersJS project or Anthropic — it's an independent tool *for* working with FeathersJS. (The `feathers-*` packages on npm are plugins for the framework, not for Claude Code.)

---

## Contents

- [What it does](#what-it-does)
- [Requirements](#requirements)
- [Installation](#installation)
- [Use with Codex and other agents](#use-with-codex-and-other-agents)
- [Verify it loaded](#verify-it-loaded)
- [Skills](#skills)
- [Agent](#agent)
- [Usage walkthrough](#usage-walkthrough)
- [How it adapts to your project](#how-it-adapts-to-your-project)
- [Troubleshooting](#troubleshooting)
- [Versioning](#versioning)
- [Contributing](#contributing)
- [License](#license)

---

## What it does

FeathersJS has a specific, opinionated structure — services split across four files, a hooks pipeline, and a resolver-based security model — that's easy to get subtly wrong. The wrong mistakes leak passwords, skip authorization, or silently 404. This plugin encodes that structure and those footguns so Claude Code generates idiomatic Feathers code and catches the common security errors before they ship.

It adds **four model-invoked skills** and **one agent**. The skills fire automatically based on what you're doing — you don't have to call them — though you can invoke them explicitly (`/feathersjs:feathers-service`, etc.).

## Requirements

- **Claude Code** with plugins enabled
- A **FeathersJS v5 (Dove)** project using **TypeScript** (the default `feathers generate app` output)
- One of the core adapters: **SQL via Knex**, **MongoDB**, or **Memory**

The plugin reads your existing code and matches its conventions, so it works with both ESM and CJS projects and either `id` (SQL) or `_id` (MongoDB) identifier styles.

## Installation

This repo is itself a Claude Code marketplace (`feathersjs-marketplace`). Install it from any Claude Code session:

```bash
/plugin marketplace add hassan4702/feathers-plugin
/plugin install feathersjs@feathersjs-marketplace
```

The install is **persistent and global** — the skills and agent become available in every project, no flags required.

**Pin to a released version** (recommended for stability — you'll only get updates when you re-pin):

```bash
/plugin marketplace add hassan4702/feathers-plugin@v0.1.1
/plugin install feathersjs@feathersjs-marketplace
```

**CLI equivalent** (outside a session):

```bash
claude plugin marketplace add hassan4702/feathers-plugin
claude plugin install feathersjs@feathersjs-marketplace
```

**Local development** — to hack on the plugin itself, clone the repo and load it directly:

```bash
git clone https://github.com/hassan4702/feathers-plugin
claude --plugin-dir ./feathers-plugin
```

## Use with Codex and other agents

The four skills follow the open [Agent Skills](https://vercel.com/docs/agent-resources/skills) standard, so they work beyond Claude Code — including **OpenAI Codex**, Cursor, GitHub Copilot, and Gemini CLI. The `feathers-expert` agent is shipped in each tool's native format.

**Skills** — install all four with the [`skills`](https://github.com/vercel-labs/skills) CLI, targeting Codex:

```bash
npx skills add hassan4702/feathers-plugin -a codex
```

Drop `-a codex` to let it auto-detect your installed agents, or target another (e.g. `-a cursor`). Skills install into Codex's skills directory (`~/.codex/skills/`, or a project's `.agents/skills/`).

**Agent (subagent)** — Claude Code reads [`agents/feathers-expert.md`](agents/feathers-expert.md) natively; for Codex, a native subagent ships at [`.codex/agents/feathers-expert.toml`](.codex/agents/feathers-expert.toml). Install it user-wide or per-project:

```bash
# user-scoped (all your projects)
mkdir -p ~/.codex/agents
curl -fsSL https://raw.githubusercontent.com/hassan4702/feathers-plugin/main/.codex/agents/feathers-expert.toml \
  -o ~/.codex/agents/feathers-expert.toml
```

Then ask Codex to use the **feathers-expert** agent for multi-file architecture or debugging questions. (Codex only spawns a subagent when you explicitly ask it to.)

## Verify it loaded

Plugin components are loaded when a session **starts**, so after installing (or updating), start a fresh Claude Code session, then:

- `/plugin` — `feathersjs` should be listed and enabled
- `/agents` — `feathers-expert` should appear
- Ask something Feathers-shaped (e.g. *"add a posts service"*) and watch a skill kick in

## Skills

### `feathers-service` — scaffold or extend a service

Generates a service using the idiomatic four-file layout and wires it up end to end:

- `<name>.class.ts` (adapter service), `<name>.schema.ts` (TypeBox + resolvers), `<name>.shared.ts` (path/methods/client types), `<name>.ts` (registration + hook wiring)
- Registers the configure function in `src/services/index.ts`, maps CRUD → HTTP verbs, reminds you about migrations for SQL adapters

**Triggers on:** "add a posts service", "create a CRUD resource/endpoint", `feathers generate service`, "why is my service 404ing".

> **Ask:** *"Add a `comments` service backed by Postgres."*
> **You get:** the four files for `comments`, a migration stub, and the `app.configure(comment)` line added to `src/services/index.ts`.

### `feathers-hooks` — write & register middleware

Covers the around / before / after / error hook pipeline used for auth, authorization, validation, logging, and side effects.

- Explains the two hook shapes (`(context, next)` vs `(context)`), the hook context (`params`, `data`, `result`, `provider`, `user`), and registration via `app.service(path).hooks({...})`
- Flags the classic pitfalls: forgetting `await next()`, mutating context by reference, mishandling internal-vs-external calls

**Triggers on:** "run a check before create", "add authentication", `HookContext`, "check permissions", "why isn't my hook firing".

> **Ask:** *"Stamp `updatedAt` before every patch on messages."*
> **You get:** a before-hook that spreads a new `updatedAt` into `context.data`, registered under `before.patch`.

### `feathers-schema` — data model, validation & security

The TypeBox schema layer plus the four resolver kinds:

- **result** (populate associations), **data** (defaults / computed / password hashing), **query** (row-level security), **external** (hide sensitive fields)
- Derives types via `Static<typeof schema>`, uses `querySyntax` and `additionalProperties: false`, keeps migrations in sync

**Triggers on:** editing `*.schema.ts`, adding/changing a field, hiding fields from responses, hashing passwords, restricting which records a user can query.

> **Ask:** *"Hide the `password` field from API responses on the users service."*
> **You get:** an external resolver setting `password` to `undefined`, wired into the service hooks.

### `feathers-review` — security-focused review

Audits Feathers code for the mistakes specific to Feathers apps, grouped **Critical → Warning → Style**:

- Missing `authenticate('jwt')`, authorization not enforced in the query resolver, secrets leaking through a missing external resolver, ownership trusted from the request body, unregistered services/hooks, schema ↔ migration drift, duplicate `$id`, unbounded `find`

**Triggers on:** "review/audit my service", "is this secure?", and proactively right after generating Feathers code.

> **Ask:** *"Review my messages service."*
> **You get:** findings by severity, each with the file, the problem, and a concrete fix — or a clean bill of health.

## Agent

### `feathers-expert`

A subagent for **multi-file architecture and debugging** questions that need codebase exploration rather than a single-file edit — service/hook/schema design decisions, "why is my endpoint 404ing", auth flows, real-time event/channel issues, adapter choices. It shows up in `/agents`. For focused single-file generation, the three building skills above are the better fit.

## Usage walkthrough

In a FeathersJS project, just describe what you want — the skills do the rest:

| You say | What happens |
| --- | --- |
| *"Add a posts service"* | `feathers-service` scaffolds the four files and registers them |
| *"Posts should belong to the logged-in user"* | `feathers-schema` stamps `userId` from `context.params.user` in the data resolver and pins reads/writes in the query resolver |
| *"Don't return the author's email"* | `feathers-schema` adds an external resolver hiding the field |
| *"Review what we just built"* | `feathers-review` audits the new service for security gaps |

## How it adapts to your project

The skills are written to **read an existing sibling service first** and match your project's conventions — `id` vs `_id`, ESM vs CJS, the database adapter, your linter — rather than imposing one layout. They prefer running the official CLI (`npx feathers generate ...`) when it's available, and reproduce its output by hand otherwise.

## Troubleshooting

- **Agent or skills don't appear after installing/updating** — plugin components load at session start. Quit and reopen Claude Code (start a fresh session), then check `/plugin` and `/agents`.
- **A new endpoint 404s** — the service was almost certainly never `app.configure(...)`-ed in `src/services/index.ts`. The `feathers-review` skill catches this.
- **A field leaks in API responses** — it's missing from the external resolver.
- **An auth check isn't enforced on reads/removes** — authorization is probably in the data resolver instead of the query resolver.

## Versioning

Releases follow [Semantic Versioning](https://semver.org); see [CHANGELOG.md](CHANGELOG.md) and the [releases page](https://github.com/hassan4702/feathers-plugin/releases). The authoritative version is the `version` field in [`.claude-plugin/plugin.json`](.claude-plugin/plugin.json) — pin to a tag (e.g. `@v0.1.1`) if you want a stable target.

## Contributing

Issues and pull requests are welcome — please [open an issue](https://github.com/hassan4702/feathers-plugin/issues) for bugs or ideas. Before submitting changes, run `claude plugin validate .` to confirm the manifest, skills, and agent still pass.

## License

[MIT](LICENSE) © Hassan. Community project; not affiliated with the FeathersJS project or Anthropic.
