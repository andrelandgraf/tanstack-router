# Neon Auth with TanStack Start

Neon Auth is Managed Better Auth: users and sessions live in the app's Lakebase
Postgres database on Neon. On Start, mount its proxy on the app's origin so the
session cookie is first-party and server functions read it per request.

A dedicated `@neon/auth` adapter for TanStack Start is not released yet. The
server below uses `@neon/auth/server` with Start's request helpers. Do not
import a TanStack Start entry point from `@neon/auth`.

For the full walkthrough, including the Router single-page-app variant, fetch
https://neon.com/docs/tanstack/auth.md.

## Env

| Variable                  | Value                                      |
| ------------------------- | ------------------------------------------ |
| `NEON_AUTH_BASE_URL`      | Branch Auth URL (written by `neon deploy`) |
| `NEON_AUTH_COOKIE_SECRET` | 32+ characters: `openssl rand -base64 32`  |

## Auth server and proxy

```ts
// src/auth.server.ts
import { createAuthServer, extractNeonAuthCookies } from '@neon/auth/server'
import { getRequest, setCookie } from '@tanstack/react-start/server'

export const authConfig = {
  baseUrl: process.env.NEON_AUTH_BASE_URL!,
  cookieSecret: process.env.NEON_AUTH_COOKIE_SECRET!,
}

export const auth = createAuthServer({
  ...authConfig,
  context: () => {
    const request = getRequest()
    return {
      getCookies: () => extractNeonAuthCookies(request.headers),
      setCookie,
      getHeader: (name) => request.headers.get(name),
      getOrigin: () =>
        request.headers.get('origin') ?? new URL(request.url).origin,
      getFramework: () => 'tanstack-start',
    }
  },
})
```

```ts
// src/routes/api/auth/$.ts
import { handleAuthProxyRequest } from '@neon/auth/server'
import { createFileRoute } from '@tanstack/react-router'
import { authConfig } from '../../../auth.server'

export const Route = createFileRoute('/api/auth/$')({
  server: {
    handlers: {
      ANY: ({ request, params }) =>
        handleAuthProxyRequest({
          request,
          path: params._splat ?? '',
          ...authConfig,
        }),
    },
  },
})
```

```ts
// src/auth.client.ts
import { createAuthClient } from '@neon/auth'

export const authClient = createAuthClient('') // same origin: /api/auth
```

## Session middleware

```ts
// src/auth.functions.ts
import { createMiddleware, createServerFn } from '@tanstack/react-start'
import {
  setResponseHeader,
  setResponseStatus,
} from '@tanstack/react-start/server'
import { auth } from './auth.server'

export const requireUser = createMiddleware({ type: 'function' }).server(
  async ({ next }) => {
    const { data } = await auth.getSession()
    if (!data?.user) {
      setResponseStatus(401)
      throw new Error('UNAUTHENTICATED')
    }
    return next({ context: { user: { id: data.user.id } } })
  },
)

export const getSessionUser = createServerFn({ method: 'GET' }).handler(
  async () => {
    setResponseHeader('Cache-Control', 'private, no-store')
    const { data } = await auth.getSession()
    return data?.user ? { id: data.user.id } : null
  },
)
```

```tsx
// src/routes/__root.tsx
export const Route = createRootRoute({
  beforeLoad: async () => ({ user: await getSessionUser() }),
  headers: () => ({ 'Cache-Control': 'private, no-store', Vary: 'Cookie' }),
})
```

Use `requireUser` on every server function that reads or writes private data,
and scope queries by `context.user.id`. Route `beforeLoad` guards stay UX only.

## Common Mistakes

### CRITICAL Trusting a user id sent by the browser

Take identity from `auth.getSession()` in middleware. A `userId` in the request
body is input to validate against the session, not proof of identity.

### HIGH Caching personalized SSR responses

Pages rendered for a signed-in user need `Cache-Control: private, no-store` and
`Vary: Cookie`, including SSR redirects, which skip the root route's `headers`.

### HIGH Keeping user-scoped client state after sign-out

Make sign-in and sign-out full page loads, so per-user `DbClient` collections
and caches are dropped with the page.

## Sources

- https://neon.com/docs/tanstack/auth.md
- https://neon.com/docs/realtime/guides/ssr
- https://neon.com/docs/auth/overview
