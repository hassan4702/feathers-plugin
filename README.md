# FeathersJS plugin for Claude Code

Helps Claude Code build [FeathersJS v5 (Dove)](https://feathersjs.com) APIs the idiomatic way — the four-file service pattern, around/before/after hooks, TypeBox schemas and resolvers — and review them for the security mistakes that bite Feathers apps.

There is no official Anthropic-maintained FeathersJS plugin; this fills that gap. (The various `feathers-*` packages on npm are plugins for the *framework*, not for Claude Code.)

## What's inside

**Skills** (model-invoked — Claude reaches for them automatically based on the task):

| Skill | Triggers on |
| --- | --- |
| `feathers-service` | Adding a resource/endpoint/CRUD service; the four-file layout; `feathers generate service`; registration/404 issues |
| `feathers-hooks` | Adding middleware; auth/authorization; `HookContext`, `next()`, before/after/around; "run before create", "check permissions" |
| `feathers-schema` | `*.schema.ts` work; TypeBox; validators; resolvers; hiding fields; populating associations; row-level security |
| `feathers-review` | Reviewing/auditing a Feathers service, hook, schema, or PR; runs proactively after generating Feathers code |

**Agent**: `feathers-expert` — for multi-file architecture and debugging questions.

## Try it locally

From this plugin's root directory:

```bash
claude --plugin-dir .
```

Then ask things like *"add a posts service"*, *"write a hook that stamps updatedAt"*, or *"review my messages service"*. Skills are invoked automatically; the agent shows up in `/agents`. Run `/reload-plugins` after edits.

## Install it (persistent)

This repo is its own marketplace (`feathersjs-marketplace`), so you can install the plugin permanently.

From a local clone:

```bash
/plugin marketplace add /Users/hassan/projects/feathers-plugin
/plugin install feathersjs@feathersjs-marketplace
```

From GitHub:

```bash
/plugin marketplace add hassan4702/feathers-plugin
/plugin install feathersjs@feathersjs-marketplace
```

## Scope & assumptions

Targets **FeathersJS v5 (Dove)** with **TypeScript**, the default generator output, and the SQL (Knex) / MongoDB / Memory core adapters. The skills tell Claude to read an existing sibling service first and match the project's conventions (id type, ESM vs CJS, adapter), so they adapt to your codebase rather than imposing one layout.

## License

MIT
