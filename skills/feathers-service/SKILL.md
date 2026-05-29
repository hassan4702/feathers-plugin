---
name: feathers-service
description: Scaffold or extend a FeathersJS v5 (Dove) service using the idiomatic four-file pattern (<name>.ts, <name>.class.ts, <name>.schema.ts, <name>.shared.ts), register it with app.configure, and wire it into the ServiceTypes declaration. Use this whenever the user wants to add a new resource, endpoint, collection, table-backed CRUD service, or "service" to a FeathersJS / Feathers app, or mentions feathers generate service, app.use, a database adapter (Knex/SQL, MongoDB, Memory), or asks why their service isn't registered. Trigger even if they just say "add a posts endpoint" or "make a CRUD resource" in a Feathers project.
---

# FeathersJS Service Scaffolding

A Feathers **service** is an object/class implementing standard CRUD methods (`find`, `get`, `create`, `update`, `patch`, `remove`, plus lifecycle `setup`/`teardown`). Services are transport-independent: the same method works over REST, websockets, or internal calls. Data-mutating methods (`create`/`update`/`patch`/`remove`) auto-emit real-time events (`created`/`updated`/`patched`/`removed`).

The official CLI (`npx feathers generate service`) produces **four files per service**. Prefer running the generator if the user has the CLI; otherwise reproduce this structure by hand. Always match the project's existing conventions (id type, database adapter, ESM vs CJS) — read a sibling service first.

## Method → HTTP mapping

| Method | HTTP | Path |
| --- | --- | --- |
| `find` | GET | `/messages` |
| `get` | GET | `/messages/:id` |
| `create` | POST | `/messages` |
| `update` | PUT | `/messages/:id` (full replace) |
| `patch` | PATCH | `/messages/:id` (merge) |
| `remove` | DELETE | `/messages/:id` |

## The four files

Replace `Message`/`message`/`messages` with the resource. `messages` is the path, `message` is the variable base, `Message` is the type. Pick the right id: SQL/Knex uses `id: Type.Number()`; MongoDB uses `_id: ObjectIdSchema()`. Read `src/declarations.ts` and an existing service to confirm.

### 1. `<name>.class.ts` — the service class

For a database-backed service, extend the adapter service rather than hand-writing CRUD. SQL/Knex example:

```ts
import type { Params } from '@feathersjs/feathers'
import { KnexService } from '@feathersjs/knex'
import type { KnexAdapterParams, KnexAdapterOptions } from '@feathersjs/knex'

import type { Application } from '../../declarations'
import type { Message, MessageData, MessagePatch, MessageQuery } from './messages.schema'

export type { Message, MessageData, MessagePatch, MessageQuery }

export interface MessageParams extends KnexAdapterParams<MessageQuery> {}

export class MessageService<ServiceParams extends Params = MessageParams> extends KnexService<
  Message,
  MessageData,
  MessageParams,
  MessagePatch
> {}

export const getOptions = (app: Application): KnexAdapterOptions => {
  return {
    paginate: app.get('paginate'),
    Model: app.get('postgresqlClient'), // or sqliteClient — match the project's db config key
    name: 'messages'
  }
}
```

For MongoDB use `MongoDBService` / `MongoDBAdapterParams` from `@feathersjs/mongodb` with `Model: app.get('mongodbClient').then(db => db.collection('messages'))`. For a non-database service, implement the methods directly on a class as shown in the Feathers services guide.

### 2. `<name>.schema.ts` — TypeBox schemas, validators, resolvers

This holds the data model, derived types, validators, and the four resolver kinds. See the `feathers-schema` skill for full detail; minimal shape:

```ts
import { resolve } from '@feathersjs/schema'
import { Type, getValidator, querySyntax } from '@feathersjs/typebox'
import type { Static } from '@feathersjs/typebox'

import type { HookContext } from '../../declarations'
import { dataValidator, queryValidator } from '../../validators'

export const messageSchema = Type.Object(
  {
    id: Type.Number(),
    text: Type.String(),
    createdAt: Type.Number()
  },
  { $id: 'Message', additionalProperties: false }
)
export type Message = Static<typeof messageSchema>
export const messageValidator = getValidator(messageSchema, dataValidator)
export const messageResolver = resolve<Message, HookContext>({})
export const messageExternalResolver = resolve<Message, HookContext>({})

export const messageDataSchema = Type.Pick(messageSchema, ['text'], { $id: 'MessageData' })
export type MessageData = Static<typeof messageDataSchema>
export const messageDataValidator = getValidator(messageDataSchema, dataValidator)
export const messageDataResolver = resolve<Message, HookContext>({
  createdAt: async () => Date.now()
})

export const messagePatchSchema = Type.Partial(messageSchema, { $id: 'MessagePatch' })
export type MessagePatch = Static<typeof messagePatchSchema>
export const messagePatchValidator = getValidator(messagePatchSchema, dataValidator)
export const messagePatchResolver = resolve<Message, HookContext>({})

export const messageQueryProperties = Type.Pick(messageSchema, ['id', 'text', 'createdAt'])
export const messageQuerySchema = Type.Intersect(
  [querySyntax(messageQueryProperties), Type.Object({}, { additionalProperties: false })],
  { additionalProperties: false }
)
export type MessageQuery = Static<typeof messageQuerySchema>
export const messageQueryValidator = getValidator(messageQuerySchema, queryValidator)
export const messageQueryResolver = resolve<MessageQuery, HookContext>({})
```

### 3. `<name>.shared.ts` — path, methods, client types

```ts
import type { Params } from '@feathersjs/feathers'
import type { ClientApplication } from '../../client'
import type {
  Message,
  MessageData,
  MessagePatch,
  MessageQuery,
  MessageService
} from './messages.class'

export type { Message, MessageData, MessagePatch, MessageQuery }

export type MessageClientService = Pick<
  MessageService<Params<MessageQuery>>,
  (typeof messageMethods)[number]
>

export const messagePath = 'messages'
export const messageMethods: Array<keyof MessageService> = ['find', 'get', 'create', 'patch', 'remove']

export const messageClient = (client: ClientApplication) => {
  const connection = client.get('connection')
  client.use(messagePath, connection.service(messagePath), { methods: messageMethods })
}

declare module '../../client' {
  interface ServiceTypes {
    [messagePath]: MessageClientService
  }
}
```

### 4. `<name>.ts` — registration + hook wiring

This is the configure function. It registers the service and attaches validators/resolvers via `schemaHooks`. **Registering hooks here is what makes validation and authorization actually run** — see the `feathers-hooks` and `feathers-schema` skills.

```ts
import { authenticate } from '@feathersjs/authentication'
import { hooks as schemaHooks } from '@feathersjs/schema'

import {
  messageDataValidator, messagePatchValidator, messageQueryValidator,
  messageResolver, messageExternalResolver, messageDataResolver,
  messagePatchResolver, messageQueryResolver
} from './messages.schema'

import type { Application } from '../../declarations'
import { MessageService, getOptions } from './messages.class'
import { messagePath, messageMethods } from './messages.shared'

export * from './messages.class'
export * from './messages.schema'

export const message = (app: Application) => {
  app.use(messagePath, new MessageService(getOptions(app)), {
    methods: messageMethods,
    events: []
  })
  app.service(messagePath).hooks({
    around: {
      all: [
        authenticate('jwt'),
        schemaHooks.resolveExternal(messageExternalResolver),
        schemaHooks.resolveResult(messageResolver)
      ]
    },
    before: {
      all: [
        schemaHooks.validateQuery(messageQueryValidator),
        schemaHooks.resolveQuery(messageQueryResolver)
      ],
      find: [],
      get: [],
      create: [schemaHooks.validateData(messageDataValidator), schemaHooks.resolveData(messageDataResolver)],
      patch: [schemaHooks.validateData(messagePatchValidator), schemaHooks.resolveData(messagePatchResolver)],
      remove: []
    },
    after: { all: [] },
    error: { all: [] }
  })
}

declare module '../../declarations' {
  interface ServiceTypes {
    [messagePath]: MessageService
  }
}
```

## Final wiring — do not skip

1. **Register the configure function** in `src/services/index.ts`: import `{ message }` and add `app.configure(message)`. A service that is never `configure`d is invisible — this is the #1 reason a new endpoint 404s.
2. **Add a migration** for SQL adapters (`npm run migrate:make -- create-messages`, then fill in `up`/`down` with `knex.schema.createTable(...)`). MongoDB needs no migration.
3. **Client registration** is handled by the `<name>.shared.ts` `messageClient` configure function if the project uses a generated client.

## Checklist before finishing

- id type (`id` number vs `_id` ObjectId) matches the project's adapter
- `$id` values are unique across the whole app (`Message`, `MessageData`, `MessagePatch`)
- The configure function is added to `src/services/index.ts`
- `authenticate('jwt')` is present unless the resource is intentionally public
- Query and data validators/resolvers are wired in the hooks object, not just defined
- A migration exists for SQL adapters
