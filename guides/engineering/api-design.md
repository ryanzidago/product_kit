# API Design

## Purpose

This guide captures reusable API design principles that apply across workspaces.

Use it when shaping:

- public HTTP APIs
- async workflow resources
- webhook contracts
- tool-calling protocols
- provider-backed runtime surfaces

Examples in this guide are illustrative. Keep project-specific policy, exceptions, and implementation history in the relevant workspace docs or artifacts.

## 1. Single by default

Public endpoints and domain-level application commands should accept one item by default.

Going from a single-item public endpoint to a batch public endpoint is backwards compatible because you add a new endpoint. Going the other way is a breaking change because you remove or change an endpoint clients already use.

The same logic mostly applies internally. Once the main domain API becomes batch-first, validation, retries, partial failure handling, idempotency, and observation semantics all start to take on batch shape. That complexity spreads quickly.

Low-level persistence helpers may still batch as an implementation detail when that improves throughput. Import jobs, backfills, and administrative tooling can also have dedicated bulk paths. The important point is that the main domain model should not become batch-first unless the product workflow is genuinely batch-first.

**Rule:** Default to single-item public endpoints and single-item domain commands. Add `POST /resources/batch` later if a real use case appears.

**Exception:** If the workflow is inherently multi-item, such as imports, mass edits, backfills, or pure CRUD operations with no side effects per item, a batch endpoint from day one can be reasonable. Even then, keep the batch path explicit instead of making it the default shape of the core API.

**Lesson learned:** A bulk `POST /messages` endpoint can look like "create 10 rows efficiently" but really mean "start 10 independent jobs." That shape drags batch-specific validation, indexed error reporting, and batch size limits into what should have been a simple single-item write path.

**Decision heuristic:**

1. If the action represents one user intent, make it single.
2. If each item can fail, retry, or complete independently, make it single.
3. If the operation starts async work or side effects, make it single.
4. If the workflow is inherently multi-item, such as import or mass edit, consider bulk.
5. If unsure, choose single.

## 2. Be honest about sync vs async

If the server does the work asynchronously, the API should say so. Do not fake synchronous behavior by blocking the HTTP request while waiting for a background job.

**Rule:** If work happens in a background job, return `202 Accepted` and give the client a way to observe the result. Do not subscribe to internal notification channels and block the request.

Observation options, in order of simplicity:

1. `GET /resource/:id` for polling. This should always be available.
2. `GET /resource/:id/await` for server-owned blocking. The client sends two requests and the server handles the wait.
3. `POST /resource/stream` or `GET /resource/:id/stream` for SSE. Use this only when live incremental output matters.

Ship the simplest option that meets the current need. More observation modes can be added later.

**Lesson learned:** A write endpoint that subscribes to internal job notifications and blocks the HTTP request until a background worker finishes is not really synchronous. The complexity exists because the API is pretending otherwise.

## 3. The server owns the waiting strategy

When clients need to wait for async work, prefer an endpoint that blocks server-side rather than making every client implement polling.

`GET /operations/:id/await` is better than "poll `GET /operations/:id` every 2 seconds" because:

- The server controls poll interval, backoff, and timeout clamping.
- The client integration is two requests, not a polling loop with retry logic.
- The server can optimize its internals later, such as moving from DB polling to PubSub or PG notify, without changing the client contract.

**Rule:** For any async creation endpoint, consider whether a corresponding `/await` endpoint eliminates client-side polling. If the resource has a clear terminal state, it probably should.

## 4. Separate creation from observation

Creating a resource and observing its progress are different concerns. They should be different endpoints, even when the client usually does both in sequence.

**Rule:** Creation endpoints return the created resource and a handle, such as `operation_id`. Observation endpoints accept that handle and return progress or results.

This allows:

- Different clients to use different observation strategies, such as poll, await, or stream.
- Creation to stay fast by persisting work and enqueueing it without waiting.
- Observation to be retried independently of creation.

Example:

```text
POST /messages           -> 202 {message, operation}      # creation
GET /operations/:id      -> 200 {operation}               # observation (instant)
GET /operations/:id/await -> 200 {operation}              # observation (blocking)
POST /messages/stream    -> SSE                           # creation + observation (future)
```

The `/stream` variant is the one case where creation and observation share a request, but it should be an explicitly different endpoint, not a flag on the creation endpoint.

## 5. Long-running workflows are durable state machines

When work spans retries, tools, human input, background jobs, webhooks, or provider round-trips, the API should expose that workflow as a durable resource with explicit status, not as a one-shot request that "usually finishes quickly."

Stripe's strongest design lesson is not just "use async." It is "model the lifecycle honestly." PaymentIntents, Invoices, and Subscriptions all behave like state machines with explicit statuses and transitions. That keeps polling, awaiting, retries, and webhook delivery anchored to one canonical object.

**Rule:** If a workflow can pause, retry, fail, resume, or require follow-up action, represent it as a durable resource with explicit statuses and allowed transitions.

That resource should:

- have a stable identifier created early
- expose its current status and terminal states
- record the inputs or outputs needed for the next step
- support multiple observation modes without changing the underlying object

In workflow-oriented systems, the run or operation should be the source of truth for execution state. Messages, tool calls, await endpoints, streams, and webhooks should all point back to the same canonical lifecycle.

## 6. Webhooks are notifications, not source of truth

Webhooks are useful, but they are delivery mechanisms, not authoritative state transitions.

Stripe's public integration pain makes this clear. Teams run into duplicate deliveries, out-of-order deliveries, stale endpoints, business logic failures after a `200`, and version mismatches between emitted events and their current request flow. Those are normal distributed-system behaviors, not rare edge cases.

**Rule:** Treat webhooks as notifications that something happened to a canonical resource. Never design a workflow where the webhook payload is the only way a client can learn the true state.

This implies:

- every webhooked workflow resource must be fetchable by ID
- webhook deliveries must have stable delivery identifiers for deduplication and tracing
- duplicate and out-of-order delivery must be considered normal
- clients should be able to reconcile against canonical resource state, not only replay event payloads

If a client can miss a webhook, replay a webhook, or process one late, the API contract should still allow it to recover from the canonical resource.

## 7. Idempotency is part of the write contract

If a client should be able to safely retry a mutating request, the API should give it an explicit idempotency mechanism.

Network failures, client timeouts, and retry middleware are normal. Without idempotency, a client cannot tell whether "request failed" means "nothing happened" or "the server did the work but the response was lost."

**Rule:** Mutating endpoints that create resources or advance workflow state should support an idempotency key or an equivalent explicit deduplication mechanism.

An idempotency key is a client-supplied token attached to a write request, usually in a header such as `Idempotency-Key`. It means: "if you receive this same operation again, treat it as the same request, not a new one."

This matters most for:

- creation endpoints, such as `POST /messages`
- state-transition actions, such as `POST /operations/:id/tool_results`
- any endpoint where repeating the same request could start duplicate work or duplicate billing

**Expected behavior:** The same idempotency key with the same request payload should return the same outcome. Reusing the key with a different payload should fail clearly.

Example:

```http
POST /messages
Idempotency-Key: 8f5c6d7e
```

If the client times out after sending the request, it should be able to retry with the same key and receive the same result instead of creating a second message or starting a second operation.

This is different from a database uniqueness constraint. A uniqueness constraint prevents duplicate stored values. An idempotency key makes repeated HTTP requests safe and gives the client a clear retry contract.

## 8. Errors are part of the API contract

Error responses should be machine-readable, stable, and traceable. They are not just strings for humans.

**Rule:** Every error response should include a stable error code and a human-readable detail message. Validation errors should preserve structured field or item information. Responses should also expose a request identifier so clients can correlate failures with server logs and support tickets.

Avoid making actionable behavior depend on parsing free-form text such as `"Unknown refs: a, b, c"`. Strings are for humans. Codes and structured fields are for software.

Minimum shape:

```json
{
  "errors": {
    "code": "operation_not_in_requires_action",
    "detail": "Operation is not in requires_action state",
    "request_id": "req_123"
  }
}
```

## 9. Separate list from search

Listing, filtering, search, and analytics are different read models. They should not be collapsed into one endpoint unless they truly have the same guarantees.

**Rule:** Use list endpoints for deterministic pagination over one stable, documented ordering. Add separate search endpoints when the semantics differ, such as text queries, ranking, fuzzy matching, alternate sort modes, or different consistency guarantees.

If search is not safe for read-after-write, say so explicitly. If analytics or export become important, they probably deserve a separate reporting path instead of overloading list or search endpoints.

Do not include total item counts in standard list responses by default. They can look helpful for UI consumers, but they often become the most expensive and least scalable part of the endpoint as datasets grow. This is especially true on distributed storage systems where filtered counts are not cheap to compute or keep fresh.

If clients genuinely need counts, treat that as a separate reporting or search concern with explicit freshness, performance, and consistency semantics instead of coupling it to every paginated list call.

Do not expose arbitrary client-controlled sorting in standard list responses by default. Sorting can look like a small UI convenience early on, but it often becomes expensive and operationally fragile as data grows. Global ordering is especially hard to provide when a list is assembled from multiple storage backends or distributed systems, and on-demand sorting can easily turn into a hot path that overloads API servers.

If clients genuinely need alternate sort modes, treat that as a separate search, reporting, or export concern with explicit constraints on supported fields, freshness, and performance rather than a generic `sort_by` on every list endpoint.

## 10. Versioning and deprecation are explicit contracts

Having `/api/v1` in the path is not enough. Clients also need to know what can change inside that version and how deprecations are handled.

**Rule:** Define what counts as a breaking change, how deprecations are announced, how long deprecated fields or endpoints remain available, and whether behavioral changes are allowed within a published API version.

Request version and event version also need to be treated as separate contracts when the API emits asynchronous payloads.

That means:

- the schema of a synchronous API response can evolve on one cadence
- the schema of emitted webhooks or events can evolve on another cadence
- upgrades need an explicit migration story for both request handling and event consumption

If clients can pin a request version, they may also need to pin or at least understand the version of emitted events. A versioning story that only covers request/response HTTP is incomplete for async systems.

Prefer additive change by default:

- add fields rather than rename them
- add endpoints rather than repurpose them
- add new observation modes rather than changing existing ones

## 11. Testability is part of the API design

An API that orchestrates async work, retries, tools, or webhooks is only usable if clients can test it deterministically.

**Rule:** When adding a workflow endpoint, decide at design time how it will be tested without production side effects.

Useful building blocks include:

- test mode or sandbox environments
- deterministic fixtures
- replayable events or webhook payloads
- duplicate-delivery simulation
- out-of-order delivery simulation
- resend or replay controls for missed deliveries
- partial-stream and interrupted-stream simulation
- upstream throttling, overload, and timeout simulation
- quota and balance exhaustion simulation
- invalid tool protocol transition fixtures, such as missing, duplicate, or out-of-order tool results
- fake clocks or time control for stateful workflows
- explicit test helpers for hard-to-trigger states

If a flow is difficult to test, that is often a sign that the API shape is still too implicit.

## 12. One preferred primitive

The API should have one clearly preferred execution primitive for each core workflow. The docs should say what that path is instead of presenting several overlapping options as equally primary.

**Rule:** For any central workflow, there should be one clearly recommended write path. Alternative paths can exist, but they should be labeled as advanced, specialized, or legacy.

This keeps the mental model small. It also prevents clients, SDKs, and internal code from drifting into several near-equivalent integration styles that all need to be supported forever.

## 13. Execution mode is orthogonal to the resource

The same core resource should support different execution or observation modes without changing the underlying object model.

**Rule:** Polling, `/await`, streaming, and webhooks should be different ways to observe or deliver the lifecycle of the same canonical resource, not separate domain models.

This matters because transport is not the domain. An operation should still be the same operation whether a client fetches it later, blocks on it, watches it over SSE, or receives a webhook when it completes.

Streaming should be treated as advisory delivery, not the only authoritative record. If a stream can be interrupted, delayed, or partially delivered, clients still need a durable resource they can fetch later to recover the final state.

## 14. State management is explicit

Clients should always know where conversation or workflow state is coming from.

**Rule:** If a workflow depends on prior state, the request should make that source of state explicit, such as inline context, a previous object handle, or a durable conversation identifier.

Do not hide state in server-side magic. If the client is operating statelessly, that should be obvious. If the client is attaching to a durable conversation, that should also be obvious. Hidden state makes retries, debugging, and migration much harder.

## 15. Scope is explicit

Clients should always know what tenant, workspace, account, project, or impersonation scope a request is operating under.

**Rule:** If a request can act on behalf of a different account, workspace, project, or tenant context, that scope must be explicit in the contract and visible in logs, traces, and emitted events.

Do not hide important scope changes in undocumented auth behavior or server-side inference. If a system has platform-owned credentials but acts in customer-specific scope, the request contract should make that boundary obvious.

This matters for:

- tenant and workspace APIs
- delegated execution
- impersonation or "act as" flows
- webhook and event payloads that need to tell clients which scope changed

Scope confusion becomes operational pain quickly. The safer default is to make acting context obvious everywhere.

## 16. Configuration is versioned separately from execution

Execution endpoints should not have to carry every behavioral choice inline forever.

**Rule:** Prompts, templates, agent profiles, tool bundles, and other reusable workflow configuration should be versioned separately from run execution whenever they are meant to be shared or evolved over time.

This creates a cleaner separation of concerns:

- configuration objects define behavior
- execution objects record what happened
- migrations can happen by changing referenced configuration rather than rewriting every caller

Not every option needs its own stored object on day one. The point is to separate reusable configuration from execution state once that configuration starts to matter across runs or clients.

## 17. Propagation rules are explicit

When one write creates, mutates, or triggers other resources, the contract should say what data propagates automatically and what does not.

A large class of API confusion comes from silent assumptions such as:

- "metadata on the parent will show up on the child"
- "the correlation ID on the create request will appear in emitted events"
- "the config referenced by the run can be reconstructed from the derived object"

**Rule:** If data can flow from one resource into another, document whether it is inherited, copied, snapshotted, transformed, or ignored.

This is especially important for:

- request metadata and correlation IDs
- tags and labels
- config references versus config snapshots
- webhook payloads derived from canonical resources
- tool-call or child-resource generation from a parent run

If propagation is selective, clients should not have to discover that experimentally.

## 18. Operational metadata is part of the contract

Operational concerns should be visible in the API contract, not hidden in logs or tribal knowledge.

**Rule:** Document and standardize the metadata clients need to operate the API safely in production, including request identifiers, provider request identifiers where relevant, optional client-supplied correlation IDs, retryability hints, `Retry-After`, rate-limit headers and their scope, whether a limit applies to requests or tokens, model or provider identifiers, usage or cost metadata where relevant, webhook delivery identifiers, and deduplication keys.

Errors already need stable codes and request identifiers. The same principle applies to successful responses and delivery systems. If a client needs a value for tracing, retry behavior, support, cost attribution, or deduplication, that value is part of the API contract.

This matters especially when different execution modes have different failure shapes. A sync request, an SSE stream, and a background job may all hit throttling differently. Clients should not have to guess whether they should retry immediately, wait for a reset window, resume an existing resource, or treat the failure as non-retryable.

## 19. Data guarantees are feature-specific

A single global privacy or retention statement is not enough for a complex API.

**Rule:** If a feature changes retention, replayability, storage duration, privacy guarantees, or observability semantics, document that at the feature level.

Examples include:

- background execution that retains state for polling
- stored responses or runs
- replayable streams or webhooks
- test helpers that persist fixtures or events

Clients should not have to infer these guarantees from implementation details.

## 20. Preferred and legacy paths are explicit

When the API evolves, the docs should clearly distinguish what new integrations should use from what older integrations may still rely on.

**Rule:** If two overlapping paths exist, document which one is preferred, which one is legacy if applicable, how to migrate, any parity gaps between the two paths, and how long the older path will remain supported.

Do not leave clients to reverse-engineer the migration story from endpoint listings alone. If we replace a workflow shape, the replacement needs a documented adoption path, not just a new endpoint.

Fast-moving APIs create real migration cost. If the new path still lacks features, has different state semantics, or changes storage behavior, that should be stated plainly instead of being discovered through forum posts or broken examples.

## 21. Upstream provider failures are normalized

If your system depends on upstream providers, those failures should not leak through as raw provider behavior.

**Rule:** Map upstream throttling, overload, insufficient balance, invalid provider state, transient transport errors, and provider-specific outages into stable error codes, statuses, and retry semantics in your own API.

Provider details can still be included as nested metadata for debugging. They should not be the only contract the client sees. The point is that clients integrate with your API semantics, not with whichever provider happened to fail underneath.

This is especially important for:

- provider rate limits versus your API's own rate limits
- provider overload or outage states
- provider account or workspace misconfiguration
- transient transport failures during streaming or webhook delivery
- provider-specific differences between initial request failure, mid-stream failure, and resumed-job failure

## 22. Tool execution is a protocol, not free-form JSON

Tool calls and tool results are not just nested blobs. They form a workflow protocol with valid state transitions.

**Rule:** Tool execution contracts must define correlation identifiers, ordering rules, valid transitions, and failure behavior. The server should validate or normalize those transitions instead of trusting clients to assemble them correctly.

That means being explicit about things like:

- every tool result referencing a prior tool call
- whether tool results must appear immediately after a tool call or can be grouped
- how parallel tool calls are correlated independently
- what happens if a tool result is missing, duplicated, or arrives out of order

If the rules are important enough to break a workflow, they belong in the API contract.

## 23. Quota and balance state are explicit

Usage limits, spend limits, prepaid balance, and workspace scope are operational state, not incidental implementation detail.

**Rule:** If a request is denied because of quota, budget, prepaid balance, subscription plan, or acting workspace, project, or tenant scope, the API should say that explicitly with stable error codes and actionable remediation.

Clients should not have to guess whether a failure means:

- retry later because of rate limiting
- add funds because prepaid balance is exhausted
- switch workspace, project, or account scope
- wait for budget reset
- contact support because the provider account is misconfigured

Operationally, these are different failures and should stay different in the contract.

## 24. Machine-readable outputs are verified, not trusted

When clients depend on structured outputs, schema compliance is a server responsibility, not just a provider promise.

**Rule:** If your API exposes machine-readable outputs from a model, validate them server-side before treating them as canonical. Parsing, schema validation, and any repair or retry behavior should be explicit and bounded.

This avoids a common failure mode where "structured output" works most of the time but occasionally emits malformed JSON, schema drift, or provider-specific edge cases that leak directly into downstream systems.

At minimum, the contract should be clear about:

- whether your API validates JSON parseability
- whether your API validates against a schema
- whether your API retries or repairs malformed output
- what error shape clients receive when structured output validation fails

## 25. Capability availability is explicit

If a capability depends on account state, provider access, configuration, feature flags, or region, the API should expose that explicitly instead of making clients infer it from unrelated failures.

**Rule:** Treat capability availability as part of the contract. Prefer explicit capability metadata, eligibility fields, or dedicated inspection endpoints over trial-and-error writes.

Examples:

- a model is visible but not enabled for the tenant
- a payment method exists but is not eligible for a given charge flow
- a tool is configured but unavailable in the selected region
- a feature is present in docs but gated by plan or rollout state

**Lesson learned:** Clients become brittle when they must distinguish “not allowed”, “not configured”, “temporarily unavailable”, and “does not exist” by probing writes and reverse-engineering error strings.

Model, transport, and feature availability should be discoverable before a client starts work, not only after a request fails.

**Rule:** If a capability depends on provider verification, preview access, organization entitlement, model family, transport mode, region, or project configuration, the API should expose that constraint explicitly or fail with a stable, actionable error before background work is enqueued.

This matters for things like:

- models that are available for sync requests but not for streaming
- features gated behind provider verification or allowlists
- capabilities that differ by workspace, project, account, or region
- modes that are supported on one provider but not another

Capability gating is not the same as quota exhaustion. "You are out of credits" and "your organization is not allowed to stream this model" are different failures and should stay different in the contract.

## 26. Delete semantics are explicit

Single-resource synchronous hard deletes should use one default public contract.

**Rule:** `DELETE /resources/:id` returns `204 No Content` on success and does not return a request or response body.

Use:

1. `204` when the resource has been synchronously removed.
2. `404` when the scoped resource does not exist.
3. `400` when scoped path params are malformed.
4. `202` only when deletion is asynchronous and the API gives clients a way to observe completion.

Do not let the HTTP contract mirror internal persistence APIs. Context or repository code may still return the deleted struct for internal follow-up work, but plain HTTP delete endpoints should not expose that as the default success shape.

If the workflow is really a reversible state transition such as archive, deactivate, or restore, model that as an explicit state change endpoint and return the updated resource with `200`.
