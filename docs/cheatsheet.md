# API Facade Cheatsheet

Condensed lookup for the API facade pattern. Each section stands on its own: one idea, one snippet, one table. For rationale, tradeoffs and bad-vs-good examples read [the handbook](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md).

**API** is anything outside your program that your program must negotiate with. A **facade** wraps it so call sites never touch transport details.

## Contents

| Area | Sections |
|---|---|
| Core | [What a facade is](#what-a-facade-is) · [Enhance fetch](#enhance-fetch-do-not-replace-it) · [Type safety](#type-safety) · [Response validation](#response-validation) · [Content type conversion](#content-type-conversion) · [Upload progress](#upload-progress) |
| Requests | [Path params](#path-params) · [Query building](#query-building) · [Sorting](#sorting) · [Filtering](#filtering) |
| Responses | [HTTP status](#http-status-handling) · [Error handling](#error-handling) · [Notifications](#notifications) · [Localization](#localization) |
| Operations | [Body mocking](#body-mocking) · [Common headers](#common-headers) · [Versioning](#api-versioning) · [Unwritten topics](#unwritten-topics) |
| Adoption | [Naming styles](#naming-styles) · [Recipes](#recipes) · [Testing](#testing-strategies) · [Migration](#migration-guide) · [Advancing](#advancing-further) |

## What a facade is

One entry point over one or more services.

| Hide | Provide |
|---|---|
| Host and static paths | `baseURL` plus path templates |
| HTTP method | one typed function per operation |
| Headers and auth | defaults set once |
| Resource versions | a single configured version |
| Response shape | stable typed result |
| Errors | typed classes with status and code |

```ts
async function getData(path: string, searchParams?: Record<string, string | number>) {
  const url = new URL(path, API_HOST)
  url.search = new URLSearchParams(searchParams).toString()

  const response = await fetch(url)
  if (!response.ok) throw new HTTPError(response.statusText, response.status)
  return response.json()
}
```

→ [README § Role](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#role)

## Enhance fetch, do not replace it

Keep `Request`, `Response` and `Headers`. Add helpers around them instead of inventing a new signature.

| Approach | Shape | Use when |
|---|---|---|
| Overshadow | `apiClient(endpoint, body?)` | small project, single backend |
| Enhance | helpers produce and consume native `fetch` types | facade will grow, edge cases need raw `fetch` |

```ts
function apiRequest(path: string, init?: RequestInit & { body?: unknown }): Request {
  const url = new URL(path, API_HOST)
  const headers = new Headers(init?.headers)
  if (init?.body && !(init.body instanceof FormData || init.body instanceof Blob)) {
    headers.set("Content-Type", "application/json")
  }
  return new Request(url, { ...init, headers, body: JSON.stringify(init?.body) })
}

function processResponse<T>(response: Response): Promise<T> {
  if (!response.ok) throw new HTTPError(response.statusText, response.status)
  if (response.status === 204) return null as T
  return response.json()
}
```

- Both approaches are valid. Neither is wrong.
- Helpers are optional. Raw `fetch` stays available.
- Do not lose `Request`/`Response`/`Headers` in the wrapper.

→ [README § Enhancing fetch](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#enhancing-fetch-instead-of-replacing-it)

## Type safety

Types are hints, not guarantees. The wire shape can differ at runtime.

| Approach | Effort | Accuracy |
|---|---|---|
| Manual types | low | drifts |
| OpenAPI + codegen | medium | good |
| Schemas as source | medium | good |
| Monorepo type import | low | exact, same repo only |

- Prefer generated types over hand-written ones when a spec exists.
- Do not mark every property nullish to silence mismatches.
- Pair types with runtime validation only where it matters.

→ [README § Type Safety](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#type-safety)

## Response validation

Validate responses against a runtime schema before returning them from the facade.

```ts
const UserSchema = z.object({ id: z.number(), name: z.string() })
type User = z.infer<typeof UserSchema>

async function fetchUser(id: number): Promise<User> {
  const payload = await (await fetch(`/api/users/${id}`)).json()
  const parsed = UserSchema.safeParse(payload)
  if (!parsed.success) throw new Error("API returned unexpected shape")
  return parsed.data
}
```

| Direction | Validate |
|---|---|
| Response | yes, where data is vital |
| Request | no, backend owns it |

- Validate in development and testing. Log in production, do not crash.
- Treat a mismatch as a backend bug, not a reason to weaken types.
- Rejecting requests on the client adds no security and blocks valid input.

→ [README § Response Validation](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#response-validation)

## Content type conversion

Let the facade convert between native objects and wire formats.

```ts
function toBody(data: any, contentType: string) {
  switch (contentType) {
    case "application/json": return JSON.stringify(data)
    case "multipart/form-data": {
      const formData = new FormData()
      for (const [key, value] of Object.entries(data)) formData.append(key, String(value))
      return formData
    }
    default: return data
  }
}
```

| Content type | Body |
|---|---|
| `application/json` | `JSON.stringify` |
| `multipart/form-data` | `FormData` |
| `blob` / binary | pass through |

→ [README § Content Type Conversion](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#content-type-conversion)

## Upload progress

Send the file as a stream, count bytes, divide by total.

```ts
let sent = 0
const tracker = new TransformStream({
  transform(chunk, controller) {
    sent += chunk.byteLength
    console.log(Math.round((sent / file.size) * 100) + "%")
    controller.enqueue(chunk)
  },
})

await fetch(url, {
  method: "POST",
  headers: { "Content-Type": file.type },
  body: file.stream().pipeThrough(tracker),
  duplex: "half",
})
```

- Resumable uploads need backend support. Do not promise them from the client alone.

→ [README § File Upload Progress](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#file-upload-progress)

## Path params

Use `URL` and a template. Never concatenate by hand.

```ts
function resolvePath(template: string, params: Record<string, string | number>) {
  const path = template.replace(/\{(\w+)\}/g, (_, key) => encodeURIComponent(String(params[key])))
  return new URL(path, API_HOST)
}

resolvePath("/users/{id}", { id: 42 }) // /users/42
```

- Always URL-encode.
- Support both `{id}` and `:id` if your backend mixes styles.

→ [README § Path Params](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#path-params-resolving)

## Query building

Build query strings with `URLSearchParams`, skip empty values.

```ts
function toQuery(params: Record<string, string | number | boolean | undefined>) {
  const search = new URLSearchParams()
  for (const [key, value] of Object.entries(params)) {
    if (value !== undefined) search.set(key, String(value))
  }
  return search
}
```

→ [README § Search Query](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#search-query-building)

## Sorting

Pick a serialization, then whitelist fields on the backend.

| Format | Query | Tradeoff |
|---|---|---|
| Stacked | `?sortBy=a&sortOrder=asc&sortBy=b&sortOrder=desc` | clear order, arrays must match |
| Array-like | `?sortBy=a,b&sortOrder=asc,desc` | compact, implicit order |
| Compact pairs | `?sort=a:asc,b:desc` | most compact, custom parse |
| Bracket | `?sort[a]=asc&sort[b]=desc` | familiar, manual encoding |

```ts
function buildSort(fields: Array<{ field: string; direction: "asc" | "desc" }>) {
  const params = new URLSearchParams()
  for (const { field, direction } of fields) {
    params.append("sortBy", field)
    params.append("sortOrder", direction)
  }
  return params
}
```

- Start with full diversity. Move to custom definitions as the API grows.
- Use stacked params for clarity, compact format when URL length matters.
- Use `qs` if it is already a dependency.
- Always validate and whitelist sortable fields on the backend.

→ [README § Sorting](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#sorting-needs-review)

## Filtering

Filters need operators, so choose a serialization up front.

| Approach | Query | Use when |
|---|---|---|
| Simple key-value | `?status=active` | equality only |
| Stacked operators | `?age__gte=18` | growing REST API |
| Record notation | `?filter[age][gte]=18` | complex filters, `qs` available |
| GraphQL | native `filters` argument | GraphQL backend |

```ts
const filters = { userId: 42, status: "active", archived: false }
const params = new URLSearchParams()
for (const [key, value] of Object.entries(filters)) params.set(key, String(value))
```

- Whitelist fields and validate operators on the backend.
- Cap filter count to avoid expensive queries.
- Document filterable fields per endpoint.
- Prefer `POST` with a body for very long filter structures.

→ [README § Filtering](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#filtering-needs-review)

## HTTP status handling

Inspect the real status. Do not collapse everything into `response.ok`.

```ts
if (response.ok) {
  if ([202, 204, 205].includes(response.status)) return null
  return response.json()
}
if (response.status === 400) throw new ClientError("Invalid request", 400)
if (response.status === 401) throw new AuthError("Unauthorized", 401)
if (response.status === 403) throw new ForbiddenError("Forbidden", 403)
if (response.status === 404) throw new NotFoundError("Not found", 404)
if (response.status === 409) throw new ConflictError("Conflict", 409)
if (response.status === 429) throw new RateLimitError("Too many requests", 429)
if (response.status >= 500) throw new ServerError("Server error", response.status)
throw new HTTPError(`Unexpected status: ${response.status}`, response.status)
```

| Status | Meaning | Facade action |
|---|---|---|
| 2xx | success | parse body, except 202/204/205 |
| 400 | bad input | `ClientError`, no retry |
| 401 | unauthenticated | refresh once, then `AuthError` |
| 403 | forbidden | `ForbiddenError`, no retry |
| 404 | missing | `NotFoundError`, may cache negative |
| 409 | conflict | `ConflictError`, retry later |
| 429 | rate limit | `RateLimitError` plus `Retry-After` |
| 5xx | server fault | `ServerError`, retry eligible |

- Use semantic codes: 201 for create, 204 for no body.
- 4xx does not retry. 5xx does.
- Add `Retry-After` to 429 responses.
- Do not duplicate status in the body.
- Log status, URL and method together.

→ [README § HTTP Status](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#http-status-handling)

## Error handling

Choose one style and keep it consistent.

| Style | Shape | Effect |
|---|---|---|
| Throw | `throw new APIError(...)` | `try`/`catch` at call sites |
| Result tuple | `[result, error]` | error handled on the spot |

```ts
enum AppErrorCode { UserNotFound = "USER_NOT_FOUND", RateLimit = "RATE_LIMIT" }
class APIError extends Error {
  constructor(public code: AppErrorCode, message: string, public status: number) {
    super(message)
  }
}
```

- Map backend error codes into an enum. Do not compare magic strings.
- Attach metadata the consumer needs: retry-after, conflicting fields, validation details.

→ [README § Error Handling](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#error-handling)

## Notifications

Facade returns codes. Consumer decides whether and how to notify.

| Situation | Notify |
|---|---|
| Expected success | no, spinner is enough |
| Long operation | optional progress then success |
| Validation error | inline at the field |
| Unexpected server error | toast |
| Session expiry | toast plus redirect |
| Optional data failure | silent, log only |

```ts
// Facade
throw new APIError(APIErrorCode.Conflict, "Conflict", 409, { conflictingFields })

// Consumer
catch (error) {
  if (error instanceof APIError) showToast(messages[error.code], "error")
}
```

- Never call `showToast()` from the facade.
- No error codes in user-facing text.
- Critical errors need persistent UI, not only a toast.
- Make messages actionable.

→ [README § Notifications](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#notifications)

## Localization

Structure comes from the facade. Translation comes from the consumer.

| Option | Where messages live | Tradeoff |
|---|---|---|
| Consumer codes | `translations/{lang}.json` | recommended, facade stays i18n-agnostic |
| Backend messages | server | backend must know every language |

```ts
const message = t(`errors.${error.code}`, error.metadata)
showToast(message, "error")
```

- Translate codes, not hard-coded strings.
- Send `Accept-Language` and keep client-side translations as fallback.
- Put numbers, field names and counts in `metadata` for pluralization.

→ [README § Localization](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#localization)

## Body mocking

Replace invalid responses with schema-generated mock data during development.

| `debug.mock` | Behavior |
|---|---|
| `"auto"` | mock only when validation fails |
| `"always"` | mock every call |

- Debug only. Never enable in production.
- Requires schemas, so it pairs with response validation.

→ [README § Body Mocking](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#body-mocking)

## Common headers

Set repeated headers once in the facade.

- Static: `Content-Type`, `Accept`, `Authorization` scheme.
- Dynamic: tokens, request IDs, session info from config.
- Correlation: generate and forward `X-Request-ID`.
- Merge consumer-provided headers over defaults.
- Provide `defaultHeaders()` plus a per-request override.

→ [README § Common Headers](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#common-headers)

## API Versioning

Pick one transport for the version and centralize it in the facade.

| Strategy | Shape | Use when |
|---|---|---|
| Path | `/v2/users` | public API, cache-friendly |
| Header | `Accept: application/vnd.api+json; version=2` | internal API, clean URLs |
| Query | `?api-version=2024-01-01` | date-based releases |

| Situation | Recommended strategy |
|---|---|
| Public API, many consumers | Path-based |
| Internal or single-consumer API | Header-based |
| Date-based release cadence | Query-param |
| Short migration window | Single facade plus versioned paths |
| Long-lived parallel versions | Versioned facade wrappers |
| Most endpoints unchanged | Adapter or mapping layer |
| Explicit cache differentiation | Path-based or query-param |
| Clean URLs at all costs | Header-based |

| Change | Breaking |
|---|---|
| Add field or endpoint | no |
| Deprecate field | no |
| Remove or rename field | yes |
| Change field type | yes |
| Remove endpoint | yes |

- Surface `Deprecation` and `Sunset` headers in development.
- Follow a timeline: deprecate, set sunset date, return 410, remove code.

→ [README § Versioning](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#api-versioning)

## Unwritten topics

These have outlines in the handbook but no full section yet. The core idea is enough to start.

| Topic | Core idea |
|---|---|
| [Caching](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#caching) | HTTP headers, ETag, 304, app cache, stale-while-revalidate, invalidate after mutations |
| [Batching](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#batching) | `POST /batch`, microtask flush, deduplicate concurrent calls |
| [Interruptions](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#interruptions) | `AbortController`, timeouts, partial streams, reconnect SSE |
| [Offline](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#offline-support--request-queuing) | `navigator.onLine`, persist queue, replay on reconnect, handle 409 |
| [Retry](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#retry-strategies) | idempotent only, 5xx and network errors, backoff with jitter, cap attempts |
| [Auth](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#authentication-patterns) | token lifecycle, single refresh then retry, API keys, OAuth, multi-tenant |

## Naming styles

Pick the surface that matches API size.

| Style | Example | Use when |
|---|---|---|
| Endpoint-based | `api.fetch.getUser({ id })` | small API, mirrors paths |
| Named | `api.user.fetch` | aliases read better |
| Resource-sliced | `api.users.create` | surface is growing |

→ [README § Naming](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#api-recipes-collection)

## Recipes

Each folder under `recipes/` is a runnable implementation.

| Recipe | Idea |
|---|---|
| [action-based](https://github.com/FrameMuse/rest-api-facade/tree/main/recipes/action-based) | calls as plain objects, pairs with reducers |
| [builder](https://github.com/FrameMuse/rest-api-facade/tree/main/recipes/builder) | fluent builder for request construction |
| [generated-methods](https://github.com/FrameMuse/rest-api-facade/tree/main/recipes/generated-methods) | one typed function per endpoint from a spec |
| [light](https://github.com/FrameMuse/rest-api-facade/tree/main/recipes/light) | minimal hand-written facade, good starting point |
| [naive](https://github.com/FrameMuse/rest-api-facade/tree/main/recipes/naive) | baseline: hard-coded URLs and `fetch` |
| [resource-sliced](https://github.com/FrameMuse/rest-api-facade/tree/main/recipes/resource-sliced) | methods grouped by resource |
| [spec-based](https://github.com/FrameMuse/rest-api-facade/tree/main/recipes/spec-based) | runtime construction from a local OpenAPI doc |

→ [README § Recipes](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#recipes)

## Testing Strategies

The facade is the seam worth testing.

- Test URL construction, path params, query strings.
- Test headers, auth and body serialization.
- Test status mapping and error classes.
- Test refresh flows and concurrent request batching.
- Mock `fetch` with `msw`, or `vi.fn()` / `jest.fn()` for simple cases.
- Do not test the `fetch` implementation itself.

No full section yet.

→ [README § Testing](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#testing-strategies)

## Migration Guide

Move incrementally from ad-hoc calls to a facade.

1. Audit endpoints, call sites, shared auth and error patterns.
2. Wrap existing code in a thin facade, migrate one resource at a time.
3. Replace hard-coded URLs, scattered `fetch` and inline errors.
4. Add validation, mocking, retry and caching.
5. Keep feature flags per endpoint for rollback.

No full section yet.

→ [README § Migration](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#migration-guide)

## Advancing Further

When the app grows, add layers around the facade.

- One facade per external API, combined behind the app model.
- A mapping layer converts wire types to native app types.
- A resource manager such as TanStack Query handles caching and mutations.
- Keep the facade copy-pasteable into another project.

→ [README § Advancing](https://github.com/FrameMuse/rest-api-facade/blob/main/README.md#advancing-further)
