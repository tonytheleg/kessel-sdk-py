# RBAC v2 Module Guidelines

This module provides RBAC workspace queries and resource/subject factory functions.
All code lives in a single file: `__init__.py`.

## Dual-Protocol Design

This module is the SDK's primary example of dual-protocol coordination:

- **REST** (`requests`): `fetch_root_workspace()`, `fetch_default_workspace()` -- query RBAC v2 workspace API
- **gRPC** (protobuf types): `list_workspaces()`, `list_workspaces_async()` -- stream workspace listings via `KesselInventoryServiceStub`
- **Pure construction**: factory functions (`principal_resource()`, `workspace_resource()`, etc.) build protobuf messages with no I/O

## REST API Conventions

### Endpoint Pattern

```text
GET {rbac_base_endpoint}/api/rbac/v2/workspaces/?type={root|default}
```

- Trailing slashes on `rbac_base_endpoint` are stripped via `rstrip('/')`.
- The `x-rh-rbac-org-id` header is required on every request.
- `Content-Type` is always `application/json`.

### Response Shape

The REST API returns `{"data": [{...}]}`. The helper takes the first item from `data`. If `data` is empty, raise `ValueError` with a message identifying the workspace type.

### Function Signatures

All REST functions follow this parameter pattern:

```python
def fetch_*(
    rbac_base_endpoint: str,
    org_id: str,
    auth: Optional[AuthBase] = None,
    http_client: Optional[requests] = None,
) -> Workspace:
```

- `auth` -- a `requests.auth.AuthBase` (from `oauth2_auth_request()`). Always pass for non-local endpoints.
- `http_client` -- injectable HTTP client (defaults to the `requests` module). Use for testing or `requests.Session` connection pooling.

### Workspace Class

`Workspace` is a plain class with four public attributes: `id`, `name`, `type`, `description`. It is not a dataclass or namedtuple. Construct from REST response dict fields.

## Factory Functions

All factory functions return protobuf message types from `kessel.inventory.v1beta2`:

| Function | Returns | Convention |
|---|---|---|
| `workspace_type()` | `RepresentationType` | resource_type="workspace", reporter_type="rbac" |
| `role_type()` | `RepresentationType` | resource_type="role", reporter_type="rbac" |
| `principal_resource(id, domain)` | `ResourceReference` | resource_type="principal", resource_id="{domain}/{id}", reporter.type="rbac" |
| `role_resource(resource_id)` | `ResourceReference` | resource_type="role", resource_id=resource_id, reporter.type="rbac" |
| `workspace_resource(resource_id)` | `ResourceReference` | resource_type="workspace", resource_id=resource_id, reporter.type="rbac" |
| `principal_subject(id, domain)` | `SubjectReference` | wraps `principal_resource()` |
| `subject(resource_ref, relation=None)` | `SubjectReference` | optional relation for subject sets |

### Resource ID Format

Principal resource IDs use `{domain}/{id}` format (e.g., `redhat/alice`). The domain is not validated -- callers provide it.

### Reporter Type

All RBAC resources use `ReporterReference(type="rbac")`. Do not use other reporter types for RBAC resources.

## Streaming Workspace Listing

### Pagination

`list_workspaces()` and `list_workspaces_async()` auto-paginate using `continuation_token`:

- Default pagination limit: 1000
- Stop when `continuation_token` is empty/falsy
- `RequestPagination` is only set when `continuation_token` is not None

### Consistency

Both functions accept an optional `Consistency` parameter, which is forwarded on every paginated request (not just the first). Tests verify this.

### Generator Pattern

- `list_workspaces()` is a sync generator (`yield`). Consume lazily with `for` or eagerly with `list()`.
- `list_workspaces_async()` is an async generator (`async yield`). Consume with `async for`.
- Neither function catches exceptions internally -- callers must handle `grpc.RpcError`.

## Testing Patterns

- REST tests: `@patch('kessel.rbac.v2.requests')` patches the module-level `requests` import.
- gRPC tests: `Mock()` the `KesselInventoryServiceStub`, with `StreamedListObjects` returning `iter([...])` (sync) or local async generators (async).
- Use `ResponsePagination(continuation_token="")` to signal end of pagination.
- Use `side_effect` with a list of iterators for multi-page pagination tests.
- Async tests require `@pytest.mark.asyncio`.

## Dependencies

This module imports directly from `kessel.inventory.v1beta2` protobuf types, from `requests`, and from `requests.auth`. The `requests` library is an optional auth extra but is unconditionally imported here -- if this becomes an issue, it would need conditional imports.
