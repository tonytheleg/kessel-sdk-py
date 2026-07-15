# Console Module Guidelines

This module extracts principal identity from Red Hat `x-rh-identity` headers
and converts them to Kessel `SubjectReference` protobuf messages.
All code lives in a single file: `__init__.py`.

## Purpose

Platform services receive an `x-rh-identity` HTTP header (base64-encoded JSON) from the Red Hat authentication proxy. This module parses that header and produces a `SubjectReference` suitable for Kessel inventory/RBAC API calls.

## Public Functions

| Function | Input | Output |
|---|---|---|
| `principal_from_rh_identity(identity, domain="redhat")` | Parsed identity dict | `SubjectReference` |
| `principal_from_rh_identity_header(header, domain="redhat")` | Raw base64-encoded header string | `SubjectReference` |

Both return a `SubjectReference` via `kessel.rbac.v2.principal_subject()`, which constructs `resource_id` as `{domain}/{user_id}`.

## Header Format

The raw header is `base64(JSON(envelope))` where the envelope has this structure:

```json
{
  "identity": {
    "type": "User",
    "org_id": "1979710",
    "user": {
      "user_id": "7393748",
      "username": "jdoe"
    }
  }
}
```

`principal_from_rh_identity_header()` handles the full pipeline: base64 decode, JSON parse, envelope unwrapping (extracting `["identity"]`), then delegates to `principal_from_rh_identity()`.

`principal_from_rh_identity()` takes the inner identity dict directly (without the `"identity"` envelope key).

## Supported Identity Types

The `_IDENTITY_TYPE_FIELDS` mapping controls which identity types are supported:

| Identity Type | Details Field | User ID Source |
|---|---|---|
| `"User"` | `"user"` | `identity["user"]["user_id"]` |
| `"ServiceAccount"` | `"service_account"` | `identity["service_account"]["user_id"]` |

All other types (`"System"`, `"X509"`, `"Associate"`, etc.) raise `ValueError`.

### Adding a New Identity Type

1. Add an entry to `_IDENTITY_TYPE_FIELDS` (e.g., `"Associate": "associate"`).
2. The user ID must be at `identity[field]["user_id"]` -- the extraction logic is shared.
3. Add test cases covering success and missing-user-id paths.

## Error Handling

All errors are `ValueError` with descriptive messages:

- `"identity must be a dict"` -- non-dict passed to `_extract_user_id()`
- `"Unsupported identity type: ..."` -- type not in `_IDENTITY_TYPE_FIELDS`
- `"Identity type {identity_type!r} is missing the {field!r} field"` -- type-specific details dict missing or not a dict
- `"Unable to resolve user ID"` -- `user_id` key absent or empty
- `"Failed to decode identity header"` -- base64 or JSON parsing failure
- `"did not decode to a JSON object"` -- decoded value is not a dict
- `"missing the 'identity' envelope key"` -- envelope lacks the `"identity"` key

Test error messages using `pytest.raises(ValueError, match=...)`.

## Default Domain

The `domain` parameter defaults to `"redhat"`. This maps to the principal resource ID format `redhat/{user_id}`. Override when the identity belongs to a different domain.

## Dependencies

This module imports from:

- `kessel.inventory.v1beta2.subject_reference_pb2` -- for the `SubjectReference` type hint and return type
- `kessel.rbac.v2.principal_subject()` -- for constructing the `SubjectReference`

It does not make network calls. It is pure parsing logic.

## Testing Patterns

- Tests are in `tests/test_console.py`.
- Use `b64encode(json.dumps(payload).encode()).decode("ascii")` to create test headers.
- Group tests by function in classes: `TestExtractUserId`, `TestPrincipalFromRHIdentity`, `TestPrincipalFromRHIdentityHeader`.
- Use `@pytest.mark.parametrize` for unsupported identity types.
- Assert on `SubjectReference` fields: `ref.resource.resource_type`, `ref.resource.resource_id`, `ref.resource.reporter.type`.
- Test realistic headers with full field sets (account_number, username, is_internal, email, etc.) to ensure extra fields are ignored.
