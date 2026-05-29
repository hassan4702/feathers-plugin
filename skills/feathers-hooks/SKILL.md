---
name: feathers-hooks
description: Write and register FeathersJS v5 hooks — pluggable middleware that runs around, before, after, or on error of a service method, for authentication, authorization, validation, logging, populating data, and side effects. Use whenever the user wants to add cross-cutting logic to a Feathers service, mentions a hook, HookContext, next(), context.params / context.data / context.result, authenticate('jwt'), "run before create", "check permissions", "log requests", or asks why a hook isn't firing or runs in the wrong order. Trigger for any Feathers middleware-shaped request even if the word "hook" isn't used.
---

# FeathersJS Hooks

Hooks are transport-independent middleware registered **around / before / after / error** of service methods, without touching the service code. They run in registration order; if one throws, the remaining hooks and the service call are skipped and the error propagates.

## Two hook shapes

**Around hooks** wrap the call and receive `(context, next)`. They must `await next()` to run the rest of the chain. Everything before `await next()` happens on the way in; everything after happens on the way out. Use these for timing, transactions, error wrapping, and the schema resolvers.

```ts
import type { HookContext, NextFunction } from '../declarations'
import { logger } from '../logger'

export const logRuntime = async (context: HookContext, next: NextFunction) => {
  const start = Date.now()
  await next()
  logger.info(`${context.method} on ${context.path} took ${Date.now() - start}ms`)
}
```

**before / after / error hooks** receive only `(context)` — no `next`. Use these for the common cases.

```ts
import type { HookContext } from '../declarations'

export const setTimestamp = async (context: HookContext) => {
  context.data = { ...context.data, createdAt: Date.now() }
  return context
}
```

A before hook that throws blocks the method. An after hook can transform `context.result`. Always return `context`.

## The hook context

Read-only: `context.app` (call other services via `context.app.service('users')`), `context.service`, `context.path`, `context.method`, `context.type`.

Writable: `context.params` — includes `context.params.query` (query filter), `context.params.provider` (`'rest'` | `'socketio'`, **`undefined` for internal calls** — use this to distinguish external from internal requests), and `context.params.user` (the authenticated user, if any). Also `context.id`, `context.data` (on create/update/patch), `context.error` (in error hooks), and `context.result` (after `await next()` or in after hooks).

## Registering hooks

Hooks are wired in the service's configure file (`src/services/<name>/<name>.ts`) via `app.service(path).hooks({...})`. The object is keyed by type, then by method, with `all` running before method-specific hooks:

```ts
app.service('messages').hooks({
  around: {
    all: [authenticate('jwt')]
  },
  before: {
    all: [],
    create: [setTimestamp],
    patch: [setTimestamp]
  },
  after: { all: [] },
  error: { all: [] }
})
```

`all` is a special key: those hooks run before the per-method ones. To run a hook only on reads: `find: [logRuntime], get: [logRuntime]`.

## Common patterns

**Authentication**: `authenticate('jwt')` from `@feathersjs/authentication`, almost always as the first `around.all` hook.

**Authorization by ownership**: prefer doing this in a *query resolver* (see `feathers-schema`) so it composes with pagination, but a before hook works for imperative checks:

```ts
export const allowOnlyOwner = async (context: HookContext) => {
  // Skip internal calls — they have no provider
  if (!context.params.provider) return context
  const user = context.params.user
  if (!user) throw new NotAuthenticated('Must be logged in')
  context.params.query = { ...context.params.query, userId: user.id }
  return context
}
```

Throw the typed errors from `@feathersjs/errors` (`BadRequest`, `NotAuthenticated`, `Forbidden`, `NotFound`, etc.) so the right HTTP status is returned.

**Calling another service**: `await context.app.service('users').get(id)` — internal calls have no `provider`, so they bypass external-only guards.

**Generate one with the CLI**: `npx feathers generate hook` (prompts for name and type). It writes to `src/hooks/<name>.ts` and you still register it in the service file.

## Pitfalls

- Forgetting `await next()` in an around hook silently skips the service call and everything after it.
- Mutating `context.data`/`context.params.query` by reference can surprise other hooks — spread into a new object.
- Registering a hook function but never adding it to the `.hooks({...})` object means it never runs.
- Order matters: put `authenticate` before authorization, and validation before resolvers that assume valid data.
- Guarding on `context.params.user` without checking `context.params.provider` will also block internal calls that legitimately have no user.
