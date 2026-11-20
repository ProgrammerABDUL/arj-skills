# Next.js as a Complete Full-Stack Architecture

Frontend, backend, database access, auth, webhooks, jobs and deployment can all live in one Next.js (App Router, TypeScript strict) codebase. This is the default architecture. Principles below are durable; **names, flags and caching APIs change between Next.js versions, so search the docs for the installed version before using any of them** (for example routing conventions, middleware/proxy file naming, caching and revalidation APIs, runtime options).

## Contents
1. Architecture and request flow
2. Which tool for which job
3. Folder structure (feature modules)
4. Data access layer and authorisation
5. Validation and error handling
6. Database
7. Caching and revalidation
8. Streaming and real-time within Next.js
9. Webhooks, uploads and background work
10. Security, rate limiting and config
11. Testing
12. Deployment and runtime limits
13. When to add a standalone service

## 1. Architecture and request flow

```
Browser
  -> Proxy/middleware (cheap checks: redirects, locale, coarse auth gate)
  -> Server Components (read data, render HTML)
  -> Client Components (interaction only)
       -> Server Actions (first-party mutations)
       -> Route Handlers (public REST, webhooks, uploads, streams)
            -> Service layer (business rules, plain TypeScript)
                 -> Data access layer (queries, authorisation)
                      -> Postgres (Neon or Supabase)
```

Rule: **route files and actions are thin.** They parse input, check identity, call a service function and shape the response. Business rules live in the service layer so they are testable and movable.

## 2. Which tool for which job

| Job | Use | Notes |
|---|---|---|
| Show data on a page | Server Component, query directly via the data layer | No API round trip needed |
| Form or button that changes data in your own UI | Server Action | Public endpoint in disguise: validate and authorise every call |
| Public or mobile-facing API, third-party access | Route Handler (REST) | Version it, document it (OpenAPI) |
| Flexible-query frontend | GraphQL endpoint in a Route Handler | Only with a stated need |
| Payment and other provider notifications | Route Handler (webhook) | Raw body, verify signature |
| File upload | Route Handler or direct-to-storage signed URL | Prefer signed URLs for large files |
| One-way live updates | Route Handler streaming (SSE) | Check platform timeout limits |
| Scheduled or deferred work | Cron trigger plus a queue/job runner | Check what the host supports |
| Cheap global checks | Proxy/middleware | Never the only authorisation layer |

## 3. Folder structure (feature modules)

```
app/
  (marketing)/ ...           public pages
  (app)/ ...                 authenticated product
  api/
    v1/ ...                  public REST route handlers
    webhooks/stripe/route.ts webhook receivers
src/
  features/
    orders/
      actions.ts             server actions (thin)
      service.ts             business rules
      queries.ts             data access
      schema.ts              zod schemas, shared types
      components/            UI for this feature
  lib/
    db/  auth/  env.ts  errors.ts  logger.ts
db/ migrations, seed
tests/ unit, integration, e2e
```

Group by feature, not by file type. One place for each rule.

## 4. Data access layer and authorisation

- Every read and write goes through functions in `queries.ts` / the data layer. Components, actions and handlers never build raw queries inline.
- Authorise inside the data layer or service (who is this user, may they touch this row), not only in the UI or proxy. Hidden buttons are not security.
- Return only the fields the caller needs (small DTOs); never pass full database rows to client components.
- Mark server-only modules so they cannot be imported into client bundles (for example with the `server-only` package; confirm current guidance).

## 5. Validation and error handling

- Validate all inputs at the boundary with Zod (or equivalent): action arguments, route handler bodies, query params, headers, webhook payloads.
- Share schemas between form and server so client and server agree.
- Typed error model: expected errors (validation, not found, forbidden, conflict) return structured results; unexpected errors are logged with context and return a safe generic message.
- Use the framework's error and not-found boundaries so every route has designed error, empty and loading states.

## 6. Database

- Postgres on Neon or Supabase with Drizzle or Prisma; migrations in the repo.
- Serverless: use the pooled connection string for runtime and the direct one for migrations; reuse the client instance across invocations as the provider's docs describe.
- Wrap multi-step writes in transactions. Keep money as integer minor units.
- Seed script for local development; preview environment per PR where the host supports it.

## 7. Caching and revalidation

- Decide per page and per query: static, revalidated on a schedule, revalidated on demand after a mutation, or fully dynamic.
- After a mutation, revalidate the affected paths or tags in the same action.
- Never cache user-specific data in a shared cache.
- Caching behaviour has changed between Next.js versions. **Read the current caching docs for the installed version before relying on defaults**, and verify behaviour in a production build, not only in dev.

## 8. Streaming and real-time within Next.js

- Loading UI and streaming Server Components for slow data.
- Server-Sent Events from a Route Handler for one-way push (notifications, progress, tickers), within host timeout limits.
- Two-way, high-frequency or long-lived connections (chat, multiplayer) need a persistent server or a managed realtime service (for example Supabase Realtime). Say so early instead of discovering it late.

## 9. Webhooks, uploads and background work

- Webhook handler: read the raw body, verify the signature, reject stale timestamps, return 2xx fast, record the event id for idempotency, process asynchronously. See `api-protocols.md`.
- Uploads: validate type and size on the server; use signed URLs to send large files straight to storage; scan or restrict what is publicly served.
- Background work: do not run long jobs inside a request. Hand off to a queue or job runner and report status through the database. Check what the chosen host supports for cron and long-running work.

## 10. Security, rate limiting and config

- Secrets only in environment variables; validate them at startup with a schema (`env.ts`) so a missing variable fails the build, not production.
- Only `NEXT_PUBLIC_*` reaches the browser; never service-role keys or database URLs.
- Security headers and a strict Content-Security-Policy configured centrally.
- Rate-limit public endpoints, login and forms (needs a shared store when running on many instances).
- Honeypot or CAPTCHA on public forms; CSRF awareness for cookie-authenticated mutations beyond what the framework provides for Server Actions (confirm current behaviour).
- CORS to named origins on public handlers.

## 11. Testing

Work test-first: each acceptance criterion from the spec becomes a failing test before the feature code is written (red, green, refactor). The layers below are where those tests live.

- Unit tests for the service layer and schemas (fast, most of the suite).
- Integration tests for data access against a real Postgres (a Neon branch or local container), not mocks, for anything involving queries or transactions.
- End-to-end tests (Playwright) for the critical journeys: sign-up, checkout, the main CRUD flow.
- Test webhook handlers with recorded provider payloads, including duplicate and out-of-order events.
- Accessibility checks and a performance budget in CI.

## 12. Deployment and runtime limits

- Know the runtime: Node runtime for full library support and database drivers; edge runtime only where its limits are acceptable.
- Know host limits: function duration, request body size, memory, cold starts. Check the host's current docs; do not assume.
- Preview deployment per pull request, production deploys from the main branch after CI passes.
- Observability from day one: structured logs, error tracking, uptime check, and a health route.

## 13. When to add a standalone service

Keep everything in Next.js until a real need appears:
- persistent WebSocket connections at scale;
- a gRPC server or other non-HTTP/1 protocol;
- jobs that run longer than the host allows;
- a workload that must scale independently of the web app.

Then add one small TypeScript service in the same monorepo, share types and schemas through a package, and keep the database as the single source of truth. Record the decision and the trade-off in a short decision record.