---
name: feathers-schema
description: Define, validate, and secure a FeathersJS v5 data model with TypeBox schemas and the four resolver kinds (result, data, query, external). Use whenever the user is working on a Feathers <name>.schema.ts file, adding or changing a field, deriving TypeScript types from a schema, validating request data or query strings, hiding sensitive fields (like passwords) from API responses, populating associations / virtual properties, hashing passwords, setting defaults like createdAt, or restricting which records a user can query. Trigger on mentions of TypeBox, Type.Object, getValidator, resolve(), querySyntax, Static, $id, "validate", "schema", or "resolver" in a Feathers context.
---

# FeathersJS Schemas & Resolvers

Schemas and resolvers are database-independent. A **schema** (TypeBox) defines shape, derives the TS type, and validates. A **resolver** transforms property values against the hook context. They are applied through the `schemaHooks` registered in the service file (see `feathers-service`).

## The four kinds — what each is for

- **Result** (`<name>Resolver`): shapes data being returned; the place to populate associations / virtual properties.
- **Data** (`<name>DataResolver`): runs on `create`/`update`/`patch` data before saving; set defaults, computed values, hash passwords, stamp `createdAt`.
- **Query** (`<name>QueryResolver`): validates/coerces the query string and is the right place to **restrict which records a user may touch** (row-level security).
- **External** (`<name>ExternalResolver`): produces the safe, outward-facing version — **hide sensitive fields here** (e.g. `password`).

## Defining the schema

```ts
import { resolve, virtual } from '@feathersjs/schema'
import { Type, getValidator, querySyntax } from '@feathersjs/typebox'
import type { Static } from '@feathersjs/typebox'
import type { HookContext } from '../../declarations'
import { dataValidator, queryValidator } from '../../validators'

export const messageSchema = Type.Object(
  {
    id: Type.Number(),            // MongoDB: _id: ObjectIdSchema()
    text: Type.String(),
    createdAt: Type.Number(),
    userId: Type.Number(),
    user: Type.Ref(userSchema)    // associated, populated by a virtual resolver
  },
  { $id: 'Message', additionalProperties: false }
)
export type Message = Static<typeof messageSchema>   // type derived from schema — single source of truth
export const messageValidator = getValidator(messageSchema, dataValidator)
```

`$id` must be globally unique. `additionalProperties: false` rejects unknown fields. Get the TS type with `Static<typeof ...>` — never hand-maintain a parallel interface.

Useful TypeBox helpers: `Type.Optional(...)`, `Type.Pick(schema, [...])`, `Type.Partial(schema)` (for patch), `Type.Intersect([...])`, `Type.Ref(otherSchema)` for associations.

## Result resolver + virtual (populate an association)

```ts
export const messageResolver = resolve<Message, HookContext>({
  user: virtual(async (message, context) => {
    return context.app.service('users').get(message.userId)
  })
})
```

`virtual()` marks a property whose value does not come from the table/collection.

## Data resolver (defaults, computed, association from auth)

```ts
export const messageDataSchema = Type.Pick(messageSchema, ['text'], { $id: 'MessageData' })
export type MessageData = Static<typeof messageDataSchema>
export const messageDataValidator = getValidator(messageDataSchema, dataValidator)
export const messageDataResolver = resolve<Message, HookContext>({
  userId: async (_value, _message, context) => context.params.user.id, // stamp owner from the JWT, never trust client
  createdAt: async () => Date.now()
})
```

Deriving `userId` from `context.params.user` (not from request body) is the correct way to prevent a client from spoofing ownership.

## External resolver (hide sensitive data)

```ts
export const userExternalResolver = resolve<User, HookContext>({
  password: async () => undefined   // never leaves the server
})
```

## Query resolver (row-level security)

```ts
export const messageQueryProperties = Type.Pick(messageSchema, ['id', 'text', 'createdAt', 'userId'])
export const messageQuerySchema = Type.Intersect(
  [querySyntax(messageQueryProperties), Type.Object({}, { additionalProperties: false })],
  { additionalProperties: false }
)
export type MessageQuery = Static<typeof messageQuerySchema>
export const messageQueryValidator = getValidator(messageQuerySchema, queryValidator)
export const messageQueryResolver = resolve<MessageQuery, HookContext>({
  userId: async (value, _user, context) => {
    // allow find-all, but force write ops to the caller's own rows
    if (context.params.user && context.method !== 'find') {
      return context.params.user.id
    }
    return value
  }
})
```

`querySyntax(props)` adds Feathers' query operators (`$gt`, `$in`, `$sort`, `$limit`, etc.) for the listed properties only — anything not listed is rejected, which is itself a security boundary.

## Password hashing

```ts
import { passwordHash } from '@feathersjs/authentication-local'
export const userDataResolver = resolve<User, HookContext>({
  password: passwordHash({ strategy: 'local' })
})
```

## Migrations follow schema changes (SQL only)

Every schema field change needs a matching Knex migration: `npm run migrate:make -- <name>`, edit `up`/`down` (`table.string(...)`, `table.bigint(...).references('id').inTable('users')`), then `npm run migrate`. MongoDB needs none.

## Pitfalls

- Adding a field to the schema but forgetting the migration → runtime/DB errors.
- Reusing a `$id` across services → validator collisions.
- Putting authorization in the data resolver instead of the query resolver → `find`/`get`/`remove` aren't protected.
- Forgetting the external resolver → passwords and secrets leak to clients.
- Defining resolvers but not wiring the matching `schemaHooks` in the service file → they never run.
