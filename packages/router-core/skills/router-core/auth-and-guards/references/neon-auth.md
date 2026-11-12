# Neon Auth with TanStack Router

Neon Auth is Managed Better Auth: users, sessions and organizations live in the
app's Lakebase Postgres database on Neon, and branch with it. This reference
covers a Router single-page app. For TanStack Start server sessions, see
`@tanstack/start-client-core#start-core/auth-server-primitives`
(references/neon-auth.md).

For the full setup, including prebuilt sign-in pages and the Start variant,
fetch https://neon.com/docs/tanstack/auth.md.

## Setup

```bash
npm install @neon/auth
neon neon-auth domain add http://localhost:3000
```

Add each deployed origin with `neon neon-auth domain add <origin>`. The branch's
Auth URL goes in `VITE_NEON_AUTH_URL`.

```ts
// src/auth.ts
import { createAuthClient } from '@neon/auth'
import { BetterAuthReactAdapter } from '@neon/auth/react/adapters'

export const authClient = createAuthClient(import.meta.env.VITE_NEON_AUTH_URL, {
  adapter: BetterAuthReactAdapter({ fetchOptions: { credentials: 'include' } }),
})
```

## Guard routes in beforeLoad

```tsx
// src/routes/_authenticated.tsx
import { createFileRoute, redirect } from '@tanstack/react-router'
import { authClient } from '../auth'

export const Route = createFileRoute('/_authenticated')({
  beforeLoad: async ({ location }) => {
    const { data: session } = await authClient.getSession()
    if (!session) {
      throw redirect({
        to: '/auth/$path',
        params: { path: 'sign-in' },
        search: { redirectTo: location.href },
      })
    }
    return { user: session.user }
  },
})
```

Concurrent `getSession()` calls are deduplicated, so calling it on each
navigation is cheap. Child routes read the non-null `user` from route context.
In components, `authClient.useSession()` gives reactive session state.

## Data access from a single-page app

A Router-only app has no server to check the session for it. To read and write
Postgres straight from the browser, use the Neon Data API with Row-Level
Security, which authorizes every request by the signed-in user's JWT:
https://neon.com/docs/tanstack/data-api.md.

## Common Mistakes

### CRITICAL Treating the Router guard as API protection

A Router-only app calls its backend directly. The backend must verify the
session on every request; `beforeLoad` only controls navigation.

### HIGH Redirecting before the session has loaded

Right after sign-up, a cached signed-out result can still be present. Await
`getSession()` in `beforeLoad`, or wait for `useSession()` to finish fetching,
before deciding the user is signed out.

## Sources

- https://neon.com/docs/tanstack/auth.md
- https://neon.com/docs/auth/quick-start/tanstack-router
