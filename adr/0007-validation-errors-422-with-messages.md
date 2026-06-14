# ADR-0007: Validation Errors Return 422 with Per-Field Messages

**Date:** 2026-06-01
**Status:** Accepted

## Context

`WebServicesManager::invParams()` and `missingParams()` returned HTTP 404 (Not Found) for validation errors. This is semantically incorrect — 404 means the resource does not exist, not that the request body is malformed.

The error response also only listed parameter names without human-readable explanations:

```json
{
  "message": "The following parameter(s) has invalid values: 'email', 'age'.",
  "type": "error",
  "http-code": 404,
  "more-info": {"invalid": ["email", "age"]}
}
```

Frontend developers could not display meaningful error messages to users without mapping parameter names to messages on the client side.

## Decision

### Status Code: 422 Unprocessable Entity

Use HTTP 422 (RFC 4918) instead of 404. Rationale:

- **400 Bad Request** — means the request is syntactically malformed (e.g., invalid JSON). Not appropriate when the syntax is correct but values fail validation.
- **422 Unprocessable Entity** — means the request is well-formed but semantically invalid. This is exactly what validation errors are.
- **404 Not Found** — means the resource doesn't exist. Completely wrong for validation.

### Response Format: Per-Field Error Messages

```json
{
  "message": "Validation failed",
  "type": "error",
  "http-code": 422,
  "more-info": {
    "errors": {
      "email": "Please provide a valid email address.",
      "age": "You must be at least 18 years old."
    }
  }
}
```

Each field maps to its error message. If no custom message is set on the parameter, an auto-generated message is used:
- Invalid: `"Invalid value for parameter 'X'."`
- Missing: `"Required parameter 'X' is missing."`

### Custom Messages

Developers set messages via `ParamOption::MESSAGE` or the `message` named argument on `#[RequestParam]`:

```php
#[RequestParam('email', ParamType::EMAIL, message: 'Please provide a valid email.')]
```

## Alternatives Considered

- **Keep 404** — incorrect semantics, confuses API consumers and monitoring tools.
- **Use 400** — too broad; doesn't distinguish between malformed syntax and failed validation.
- **Only fix status code, no messages** — still poor DX for frontend developers.
- **Structured error codes instead of messages** — more machine-friendly but requires a code registry; messages are simpler and immediately useful.

## Consequences

### Breaking Change

- HTTP status code changes from 404 to 422 for validation errors.
- Response `more-info` structure changes from `{"invalid": [...]}` / `{"missing": [...]}` to `{"errors": {"field": "message"}}`.
- Clients checking for 404 on validation errors must update to 422.

### Migration

```diff
- if (response.status === 404 && response.body.type === 'error') {
+ if (response.status === 422) {
-     // handle validation error from 'more-info.invalid' or 'more-info.missing'
+     // handle validation error from 'more-info.errors'
+     Object.entries(response.body['more-info'].errors).forEach(([field, message]) => {
+         showFieldError(field, message);
+     });
  }
```

### Benefits

- Correct HTTP semantics — monitoring tools and API gateways can distinguish "not found" from "validation failed".
- Frontend developers get displayable error messages without client-side mapping.
- Per-field errors enable inline form validation UX.

## GitHub Issue

[webfiori/http#113](https://github.com/WebFiori/http/issues/113)
