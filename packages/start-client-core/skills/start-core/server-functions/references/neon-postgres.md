# Postgres on Neon with TanStack Start

Neon is a set of cloud backend primitives built around Lakebase Postgres,
including Auth, Realtime and Object Storage. In a Start app, all database
access stays behind `createServerFn` or server routes.

Keep the app's existing database, ORM and driver. These steps apply to a new
app or one already on Neon.

## Get a connection string

- Existing Neon project: copy the connection string from the Console's
  **Connect** dialog.
- No account yet: `npx neon@latest claim create` provisions a claimable
  project and writes `DATABASE_URL` and `DATABASE_URL_UNPOOLED` to an existing
  `.env`, otherwise `.env.local`. A person can claim it into their Neon account
  before it expires.

Use the pooled connection string (`-pooler` in the host) for app traffic and
the direct one for migrations and session-level features such as `LISTEN` or
advisory locks.

## Query from a server function

```ts
// src/db.server.ts
import { Pool } from 'pg'

export const pool = new Pool({ connectionString: process.env.DATABASE_URL })
```

```ts
// src/todos.functions.ts
import { createServerFn } from '@tanstack/react-start'
import { pool } from './db.server'

export const listTodos = createServerFn({ method: 'GET' })
  .middleware([requireUser])
  .handler(async ({ context }) => {
    const { rows } = await pool.query(
      'SELECT id, title, completed FROM todos WHERE user_id = $1',
      [context.user.id],
    )
    return rows
  })
```

For the Drizzle setup (schema, `drizzle.config.ts`, migrations, server
functions), fetch https://neon.com/docs/tanstack/postgres-drizzle.md.

## Pick the driver for the deploy target

| Runtime                                           | Driver                                                                  |
| ------------------------------------------------- | ----------------------------------------------------------------------- |
| Node server, Vercel Fluid compute, Neon Functions | `pg` (`Pool`), created once at module scope                             |
| Cloudflare Workers                                | `pg` through Cloudflare Hyperdrive                                      |
| Other serverless or edge functions                | `@neondatabase/serverless` (`neon()` over HTTP, `Pool` over WebSockets) |

Details and Drizzle imports per driver:
https://neon.com/docs/tanstack/drivers.md.

## Branches for migrations and previews

A Neon branch is a copy-on-write copy of the database, so creating one does not
copy data and takes the same time for any database size.

```bash
neon branches create --name preview/pr-123
neon connection-string preview/pr-123 --pooled
```

- Run new migrations against a branch with production data before applying
  them to the production branch.
- Give each preview deployment its own branch. Neon's Vercel integration
  creates one per preview automatically.

## Add only what the app needs

- Login and sessions: Neon Auth. See `start-core/auth-server-primitives`
  (references/neon-auth.md).
- File uploads: Neon Object Storage, S3-compatible buckets that branch with the
  database. Sign upload URLs in a server function:
  https://neon.com/docs/tanstack/object-storage.md.
- Live query results in the UI: Neon Realtime with TanStack DB. See
  `@tanstack/db#db-core/collection-setup` (references/neon-realtime-adapter.md).
- Ordinary reads and writes need none of these.

## Sources

- https://neon.com/docs/tanstack/postgres-drizzle.md
- https://neon.com/docs/tanstack/drivers.md
- https://neon.com/docs/tanstack/object-storage.md
- https://neon.com/docs/connect/choose-connection
