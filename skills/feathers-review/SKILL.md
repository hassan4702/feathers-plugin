---
name: feathers-review
description: Review FeathersJS v5 code for correctness, security, and idiomatic structure before it ships. Use whenever the user asks to review, audit, or sanity-check a Feathers service, hook, schema, or PR, or asks "is this secure / is this right", or right after generating or editing Feathers code. Focuses on the things that bite Feathers apps — missing authentication, authorization that isn't enforced in query resolvers, sensitive fields leaking through missing external resolvers, unregistered services/hooks, and schema/migration drift. Trigger proactively after writing Feathers code even if review wasn't explicitly requested.
---

# FeathersJS Code Review

Walk the changed/affected files and check the items below. Report findings grouped by severity (Critical → Warning → Style), each with the file, the problem, and a concrete fix. Don't just restate the code — say what's wrong and what to change.

## Critical (security / correctness)

1. **Authentication present.** Each non-public service should have `authenticate('jwt')` in `around.all`. Flag any service exposed externally (`methods` includes mutating verbs) without it. Confirm "public" services are public on purpose.
2. **Authorization actually enforced.** Ownership/tenancy checks belong in the **query resolver** (`<name>QueryResolver`) so they cover `find`/`get`/`patch`/`remove`, not only in a data resolver (which misses reads/removes). Verify writes are pinned to `context.params.user`, not to a client-supplied id.
3. **Sensitive fields hidden.** Every schema with secrets (`password`, tokens, internal ids) needs an **external resolver** setting them to `undefined`. A missing external resolver leaks them in API responses.
4. **Ownership derived from auth, not body.** `userId`/`ownerId` should be set in the data resolver from `context.params.user`, never trusted from `context.data`.
5. **Service is registered.** The configure function must be `app.configure(...)`-ed in `src/services/index.ts`. An unregistered service silently 404s.
6. **Hooks are wired.** A hook/resolver that's defined but not added to the `.hooks({...})` object never runs — check each validator/resolver is referenced.
7. **`additionalProperties: false`** on data schemas, so unexpected fields are rejected rather than written.
8. **Query surface is minimal.** `querySyntax(...)` should list only properties safe to filter/sort on; sensitive or expensive columns shouldn't be queryable.

## Warning (robustness)

- **`await next()` in around hooks** — missing it skips the service call and all downstream hooks.
- **Internal vs external calls** — guards that read `context.params.user` should also consider `context.params.provider` (`undefined` = internal) so server-side calls aren't wrongly blocked or wrongly trusted.
- **Typed errors** — throw `@feathersjs/errors` (`BadRequest`, `NotAuthenticated`, `Forbidden`, `NotFound`) rather than generic `Error`, so HTTP status codes are correct.
- **Schema ↔ migration drift** (SQL) — every new/changed/removed field needs a matching Knex migration with a working `down`. Flag schema edits with no migration.
- **Unique `$id`** — duplicated schema `$id` values collide in the validator registry.
- **Pagination** — large collections should set/respect `paginate`; unbounded `find` is a DoS/perf risk.
- **By-reference mutation** of `context.data` / `context.params.query` — spread into new objects.

## Style / idiom

- Four-file layout per service (`.ts`, `.class.ts`, `.schema.ts`, `.shared.ts`); logic in the expected file.
- Types derived via `Static<typeof schema>`, not hand-written parallel interfaces.
- Reuse `all` hooks instead of repeating the same hook across every method.
- Match the project's existing conventions (id type, ESM/CJS, db adapter, linter) — read a sibling service to confirm before suggesting changes.

## Output format

```
## Review: <files>

### Critical
- [services/messages/messages.ts] No authenticate('jwt') in around.all — the messages
  service is writable over REST by anyone. Add it as the first around.all hook.

### Warning
- [services/messages/messages.schema.ts] userId added to schema but no migration found.
  Add a Knex migration adding the column with a foreign key to users.id.

### Style
- ...

### Looks good
- External resolver correctly hides password.
```

If everything checks out, say so plainly rather than inventing issues.
