# Auth Module Guidelines

This module implements OAuth 2.0 Client Credentials flow for Kessel SDK authentication.
It is an optional dependency -- installed via `pip install "kessel-sdk[auth]"`.

## Editable Files

Only two files in this directory are hand-written:

- `__init__.py` -- public API exports (`__all__`)
- `auth.py` -- all auth logic

## Architecture

### Class Hierarchy

- `OAuth2ClientCredentials` -- core token manager (client credentials grant via `BackendApplicationClient`)
- `GoogleOAuth2ClientCredentials(google.auth.credentials.Credentials)` -- adapter for gRPC auth metadata injection via `google.auth.transport.grpc.AuthMetadataPlugin`
- `AuthRequest(requests.auth.AuthBase)` -- adapter for `requests` library HTTP auth
- `RefreshTokenResponse` -- return type from `get_token()` (access_token + expires_at)
- `OIDCDiscoveryMetadata` -- thin wrapper around the OIDC discovery JSON document

### Factory Functions

- `fetch_oidc_discovery(issuer_url)` -- fetches `/.well-known/openid-configuration`, returns `OIDCDiscoveryMetadata`
- `oauth2_auth_request(credentials)` -- creates `AuthRequest` for use with `requests` library

## Conventions

### Token Lifecycle

- `get_token()` caches tokens and auto-refreshes 300 seconds before expiry. Do not reduce this buffer below 60 seconds.
- Missing `expires_in` in the token response defaults to 0, causing immediate re-fetch on next call.
- `force_refresh=True` bypasses the cache and forces a new SSO call.

### Thread Safety -- Double-Checked Locking

`get_token()` uses a `threading.Lock` with a `_generation` counter to coalesce concurrent refresh requests into a single SSO call. The pattern:

1. Snapshot `_generation` before acquiring the lock.
2. After acquiring, if `_generation` changed and the token is valid, skip the SSO call.
3. After fetching, increment `_generation`.

When modifying `get_token()`, preserve this pattern. Tests in `test_auth.py` verify that 20 concurrent threads produce exactly one SSO call.

### Datetime Handling

All internal datetime comparisons use naive UTC (`datetime.now(timezone.utc).replace(tzinfo=None)`). Do not mix aware and naive datetimes -- the `_expiry` field is always naive UTC.

### OIDC Discovery

- `fetch_oidc_discovery()` strips trailing slashes from the issuer URL before appending the well-known path.
- The function sets `timeout=10` on the HTTP request. Do not remove this timeout.
- Always call `response.raise_for_status()` before parsing JSON.

### Separation of Concerns

`OAuth2ClientCredentials` only accepts a direct `token_endpoint` URL. OIDC discovery is a separate step (`fetch_oidc_discovery()`). Do not merge them.

## Import Guard

This module depends on optional extras (`google-auth`, `requests-oauthlib`, `requests`). Other SDK modules that reference auth types must use `TYPE_CHECKING` guards:

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from kessel.auth import OAuth2ClientCredentials
```

Runtime imports of auth modules belong inside function bodies (see `kessel/grpc/__init__.py`).

## Public API

`__init__.py` re-exports exactly these names via `__all__`:

- `OAuth2ClientCredentials`
- `GoogleOAuth2ClientCredentials`
- `OIDCDiscoveryMetadata`
- `fetch_oidc_discovery`
- `oauth2_auth_request`

When adding a new public symbol, add it to both `auth.py` and `__init__.py`'s `__all__`.

## Testing Patterns

- Patch `kessel.auth.auth.requests.get` for OIDC discovery tests (patch at import site).
- Use `patch.object(credentials._session, "fetch_token", ...)` for token fetch tests.
- Use `Mock()` only -- not `MagicMock`, `AsyncMock`, or `create_autospec`.
- Test concurrent refresh behavior with `threading.Barrier` and `ThreadPoolExecutor`.
- All test credentials must be obviously fake (`"test-client-id"`, `"https://example.com/token"`).

## Security Rules

- Never log or print tokens, client secrets, or authorization headers.
- Never provide non-empty defaults for credential env vars -- always default to `""`.
- Client Credentials grant only (`BackendApplicationClient`). Do not introduce authorization code or implicit flows.
