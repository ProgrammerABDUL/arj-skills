# API Protocols Playbook

Read this before designing or building any API or integration. The principles here are durable; exact syntax, SDK calls and provider details are not, so **search current official docs before writing code** (see the search-first rule in SKILL.md).

## Contents
1. REST
2. GraphQL
3. WebSockets
4. gRPC
5. SOAP
6. Webhooks
7. Payments checklist
8. Combining styles

## 1. REST (default: web apps, mobile apps, public APIs)

- Resource-based URLs (`/orders/{id}`), correct verbs, correct status codes. Plural nouns, no verbs in paths.
- Describe the API with an OpenAPI spec and generate TypeScript types or clients from it.
- One consistent error shape (for example RFC 9457 problem details) with a stable machine-readable code.
- Cursor-based pagination for large or changing lists; filters and sort as query parameters.
- Idempotency keys on unsafe operations that must not run twice (payments, order creation).
- Version public APIs (`/v1`) and never break a published version. Mobile apps lag behind releases in app stores, so keep old versions alive long enough.
- Rate limiting, auth (session cookie for first-party web, OAuth2 or API keys/tokens for third parties and mobile), CORS to named origins.
- Document every endpoint with an example request and response.

## 2. GraphQL (complex frontends needing flexible data without over-fetching)

Use it when many screens need different slices of related data, or several clients need different shapes. Do not use it for simple CRUD.

- Schema-first design; generate types for client and server.
- Prevent N+1 resolver queries with batching (DataLoader pattern).
- Protect the server: query depth and complexity limits, timeouts, pagination on every list, persisted queries for public clients.
- Authorise at the resolver or field level, not only at the gateway.
- Disable or restrict introspection and playground in production for private APIs.
- Errors: partial data plus an `errors` array is normal; design client handling for it.
- Caching is harder than REST; plan it (client cache normalisation, persisted queries, CDN for public queries).

## 3. WebSockets (live chat, multiplayer games, real-time tickers)

- Needs a **persistent server**. Serverless platforms that kill idle functions (Vercel functions, for example) do not hold WebSocket connections. Options: a standalone Node service, or a managed realtime layer (Supabase Realtime, Ably, Pusher and similar). Name this trade-off.
- If updates only flow server to client, consider Server-Sent Events: simpler, proxy-friendly, auto-reconnect.
- Authenticate at the handshake (short-lived token), authorise every subscription and message.
- Define a message schema with a type and version field; validate every inbound message.
- Heartbeats (ping/pong) to detect dead connections. Client reconnects with exponential backoff and jitter, then resyncs missed messages (last-event id or sequence number).
- Scaling out: multiple instances need a pub/sub backbone (Redis or similar) so a message reaches clients connected to other instances.
- Limit message size and rate per connection; apply backpressure rather than buffering without bound.
- Games and tickers: decide the authority (server-authoritative for games), tick rate and what to do on packet loss.
- Accessibility: announce live updates with ARIA live regions without flooding screen readers.

## 4. gRPC (internal microservices, low-latency distributed systems)

- Protobuf contracts are the source of truth. Lint and detect breaking changes in CI (for example with Buf).
- Evolve contracts safely: add fields, never reuse or renumber existing field numbers, reserve removed ones.
- Always set deadlines and propagate them; retry only idempotent calls, with backoff.
- Use mTLS between services; propagate auth identity in metadata.
- Streaming (server, client, bidirectional) for large or continuous data.
- Browsers cannot speak native gRPC. If a browser must call it, use gRPC-Web or Connect, or put a REST/GraphQL gateway in front.
- Do not split into microservices just to use gRPC. One service means REST or direct function calls are simpler.
- Add health checks, structured logging and tracing (OpenTelemetry) across calls.

## 5. SOAP (only when a bank or payment gateway requires it)

Never design a new system around SOAP. Integrate with it because a partner demands it.

- Get the provider's official WSDL, docs and sandbox credentials. Do not guess endpoints, field names or namespaces.
- Generate or use a maintained SOAP client for the language; verify the current library before choosing.
- Put an **adapter layer** around it: the rest of the app calls clean typed functions (`createPayment(input)`) and never sees XML.
- Handle WS-Security, certificates and IP allow-lists as the provider specifies; keep keys and certificates out of the repo.
- Timeouts on every call. Retry only idempotent operations; use reference numbers from the provider to reconcile uncertain results.
- Log requests and responses for audit with sensitive fields (card numbers, credentials) masked.
- Test in the provider's sandbox first, with their test cases; get explicit user approval before any live call.

## 6. Webhooks (payment notifications such as Stripe, automated triggers)

Receiving:
- **Verify the signature** using the provider's method, on the **raw request body** (parsing JSON first can break verification). Reject stale timestamps to stop replays.
- Return a 2xx quickly, then process asynchronously (queue or background job). Slow handlers cause provider retries.
- **Idempotency:** store processed event ids and ignore duplicates. Providers deliver at least once.
- Do not rely on ordering; handle events arriving late or out of order. For critical state (paid, refunded), re-fetch the object from the provider API instead of trusting the payload alone.
- Separate signing secrets per endpoint and environment; rotate when leaked.
- Log every event id, type and outcome. Alert on repeated failures. Provide a way to replay events.
- Test locally with the provider's tooling (for example the Stripe CLI) and in test mode before live.

Sending (your own webhooks):
- Sign payloads, include an event id and timestamp, retry with exponential backoff, keep a dead-letter record, and document the format and verification steps.

## 7. Payments checklist

- Use hosted checkout or the provider's secure elements. Never store or log raw card data.
- Test mode and live mode keys are separate; the live key never appears in code, logs or client bundles.
- Fulfil orders from verified webhook events, not from the browser redirect alone.
- Reconcile your records with the provider regularly; store provider ids for every payment, refund and dispute.
- Handle failure, retry and refund states in the UI, not only success.
- Never invent fees, limits or supported countries; read the provider's current docs.

## 8. Combining styles

- Public API for third parties: REST (plus webhooks for events).
- Complex dashboard frontend: GraphQL or REST with well-designed endpoints; choose one primary style per client.
- Live features: WebSocket or managed realtime beside the main API; keep the source of truth in the database.
- Internal services: gRPC only once there are multiple services with real latency or contract needs.
- Legacy partner: SOAP adapter at the edge, nothing else touches it.
- Every style: validate input, authenticate, rate-limit, log with context, document, version.