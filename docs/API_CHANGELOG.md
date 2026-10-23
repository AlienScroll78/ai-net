# API Changelog

This document records public REST API changes found in the repository history and explains the API compatibility policy. Entries are in reverse chronological order.

> The repository has no API release tags. Dates below are the dates of the source commits, not independently verified production release dates. Version status and configured sunset dates are taken from the `/api/versions` runtime manifest. Confirm deployment-specific configuration before planning a migration.

## Versioning Policy

The API uses semantic versioning for contract compatibility:

| Version change           | When to use it                                                                          |
| ------------------------ | --------------------------------------------------------------------------------------- |
| Major (`1.x` to `2.0`)   | A change that can make an existing client request fail or require client code changes.  |
| Minor (`1.0` to `1.1`)   | Backward-compatible additions, such as optional fields, endpoints, or query parameters. |
| Patch (`2.0` to `2.0.1`) | Backward-compatible fixes that do not change the documented contract.                   |

Breaking changes include removing or renaming a field or endpoint, changing a field's type or meaning, changing required authentication, changing a request method, or changing a pagination or response format in a way existing clients cannot handle. Adding optional response fields, endpoints, or accepted enum values is additive; clients should ignore unknown response fields and enum values.

### Deprecation and Sunset

The runtime version manifest at `GET /api/versions` specifies at least six months' deprecation notice and a twelve-month sunset grace period. Deprecations and planned sunset dates are listed below and in that endpoint; `API_V1_SUNSET_DATE` can override the default sunset date. Deployments should publish the `Deprecation` and `Sunset` response headers when applicable.

The intended sunset behavior is to stop serving a retired version and return `410 Gone` with a migration reference. The current versioning middleware advertises deprecated versions and validates supported versions, but does not itself enforce that `410 Gone` behavior. Check the runtime manifest and deployment behavior rather than treating the policy as proof that a version has already been removed.

### Pinning a Version

Send the `API-Version` request header. The server returns the negotiated value in `X-API-Version`; if omitted, the default is `1.0` unless the deployment configures another value. The current supported values are configured by `API_SUPPORTED_VERSIONS` (defaults to `1.0,1.1,2.0`). Version selection is header-based, not a `/api/v1` or `/api/v2` URL prefix.

```http
GET /api/tasks HTTP/1.1
Host: api.example.com
API-Version: 2.0
```

For example, pin a client to the v1 contract with:

```bash
curl -H 'API-Version: 1.0' https://api.example.com/api/tasks
```

An unsupported version currently returns `400` and lists supported versions.

## Changelog

### 2026-09-01 — API versions 1.0 and 2.0 — Added

- Task submission accepts the `Idempotency-Key` request header. Reuse the same key when retrying a submission to avoid creating a duplicate task. The key is optional; requests without one retain the existing behavior.

### 2026-08-31 — API versions 1.0, 1.1, and 2.0 — Added

- Trace identifiers are propagated across REST, agent, and payment operations. Use the returned request or trace identifiers to correlate a request with diagnostics.

### 2026-08-31 — API versions 1.0, 1.1, and 2.0 — Changed

- Added token-bucket rate limiting and a structured error schema. Clients receiving `429` should respect the `Retry-After` header when present; clients should branch on stable error codes rather than matching message text.

### 2026-08-30 — Unversioned health endpoints — Added

- Added `/health/ready` and `/health/live` probes alongside `/health` for readiness, liveness, and dependency health checks.

### 2026-08-29 — API versions 1.0 and 2.0 — Added

- Added cursor-based pagination for task, event, and agent listings. The task v1 route retains its offset-based response; v2 supports cursor pagination when `cursor` or `limit` is supplied. Agent listings also retain the legacy flat response when cursor parameters are omitted.

### 2026-08-25 — API versions 1.0, 1.1, and 2.0 — Added

- Added header-based version negotiation using `API-Version`, with the selected version echoed in `X-API-Version`. The default remains `1.0` for backward compatibility; v2 task routes can be selected by sending `API-Version: 2.0`.

### 2026-07-23 — Task endpoints — Changed

- Added ownership checks to task detail and deletion operations. A caller must provide the wallet public key associated with the task; unauthorized access is rejected instead of returning the task or deleting it.

## Deprecations

The following statuses and dates come from the runtime `/api/versions` manifest. The v1 sunset defaults to `2027-01-01` and may be overridden by deployment configuration.

| Version | Status     | Deprecated   | Planned sunset          | Migration target |
| ------- | ---------- | ------------ | ----------------------- | ---------------- |
| `1.0`   | Deprecated | `2026-01-01` | `2027-01-01` by default | `2.0`            |
| `1.1`   | Deprecated | `2026-06-01` | `2027-01-01` by default | `2.0`            |

No removed field or endpoint with a separately announced removal date is identifiable in the checked repository history. Do not treat the dates above as a deprecation of individual fields or routes.

## Migration Examples

### Pinning from the default version to v2

The version is selected with a header. Do not rewrite the path to `/api/v2`:

```bash
# Before: uses the configured default, which is 1.0 by default
curl https://api.example.com/api/tasks

# After: explicitly pins the request to v2
curl -H 'API-Version: 2.0' https://api.example.com/api/tasks
```

### Task submission response envelope

The v1 task submission response is a top-level object; v2 wraps the same task fields in `data` and adds metadata and links.

```json
// v1.0
{
  "taskId": "task_abc123",
  "dagPreview": [],
  "status": "queued"
}
```

```json
// v2.0
{
  "data": {
    "taskId": "task_abc123",
    "dagPreview": [],
    "status": "queued"
  },
  "_meta": {
    "version": "2.0",
    "apiVersion": "2.0"
  },
  "_links": {
    "self": "/api/tasks/task_abc123",
    "stream": "/api/tasks/task_abc123/stream"
  }
}
```

Move reads of `taskId`, `dagPreview`, and `status` under `data` when opting into v2. Metadata can also be used to confirm the response version.

### Task-list pagination

The v1 task list uses offset parameters and returns a top-level `tasks` array. In v2, pass a cursor (or an initial `limit`) and read items and the next cursor from the `data` envelope:

```http
GET /api/tasks?page=1&pageSize=20
API-Version: 1.0
```

```http
GET /api/tasks?limit=20
API-Version: 2.0
```

```json
{
  "data": {
    "items": [],
    "pagination": {
      "limit": 20,
      "nextCursor": "opaque-cursor",
      "hasNextPage": true
    }
  }
}
```

For the next page, send the returned cursor with the same limit and API version:

```http
GET /api/tasks?cursor=opaque-cursor&limit=20
API-Version: 2.0
```

## Runtime Alignment Note

The runtime manifest currently describes v1.1 task-result envelope and v2 agent-registration changes in addition to the task contract changes shown above. Those descriptions are not consistently represented by the active route handlers, which negotiate versions specifically for task routes. Treat those manifest summaries as pending API-team confirmation before changing clients; keep this changelog aligned with the deployed OpenAPI contract as those routes are reconciled.

## Related Documentation

- [REST API Reference](./API_REFERENCE.md)
- [Migration Guide](./MIGRATION_GUIDE.md)
