# ADR-0005: Replace WebServicesManager with RequestProcessor

**Date:** 2026-06-01
**Status:** Accepted

## Context

`WebServicesManager` is a god class that combines service registry, request dispatching, parameter filtering, auth orchestration, error responses, content type parsing, PUT body parsing, output stream management, and OpenAPI generation.

This creates several problems:
- Every new service requires touching two files (service + manager)
- The manager class is ~600 lines of mixed concerns
- Testing requires instantiating the full manager even for a single service
- The class is hard to extend or override partially

The `WebService` class already carries all the metadata needed via attributes (`#[RestController]`, `#[GetMapping]`, `#[RequestParam]`, `#[PreAuthorize]`). The manager is mostly orchestration glue.

## Decision

Introduce a `RequestProcessor` class that takes a `WebService` instance and a `Request`, and produces a `Response`. Split the manager's responsibilities into focused components:

```php
class RequestProcessor {
    public function process(WebService $service, Request $request): Response;
}
```

The processor handles:
1. Match HTTP method → annotated method
2. Filter/validate parameters
3. Check authorization (annotations)
4. Call the method
5. Serialize return value
6. Return Response (or error response on failure)

Other responsibilities are extracted:
- PUT/PATCH body parsing → `Request` class
- OpenAPI generation → standalone `OpenAPIGenerator`
- Error responses → `ErrorResponse` helper class

`WebServicesManager` is preserved but deprecated. It becomes a thin wrapper: registry + delegates processing to `RequestProcessor`.

## Implementation Steps

1. **Move PUT/PATCH body parsing into `Request`** — `Request` should parse its own body regardless of HTTP method
2. **Extract `ErrorResponse` helper** — static methods for standardized JSON error responses
3. **Extract `OpenAPIGenerator`** — accepts array of `WebService` instances, generates spec
4. **Create `RequestProcessor`** — core orchestrator (method matching, param filtering, auth, invocation, serialization)
5. **Deprecate `WebServicesManager`** — mark as `@deprecated`, delegate `process()` to `RequestProcessor` internally

Each step is independently testable and shippable. No step breaks backward compatibility.

## GitHub Issues

| Step | Issue |
|------|-------|
| 1 | [webfiori/http#118](https://github.com/WebFiori/http/issues/118) — Move PUT/PATCH parsing into Request |
| 2 | [webfiori/http#119](https://github.com/WebFiori/http/issues/119) — Extract ErrorResponse helper |
| 3 | [webfiori/http#120](https://github.com/WebFiori/http/issues/120) — Extract OpenAPIGenerator |
| 4 | [webfiori/http#121](https://github.com/WebFiori/http/issues/121) — Create RequestProcessor |
| 5 | [webfiori/http#122](https://github.com/WebFiori/http/issues/122) — Deprecate WebServicesManager |

## Alternatives Considered

- **Keep the manager, just add auto-discovery** — reduces boilerplate but doesn't address the god class problem.
- **Remove the manager entirely** — cleaner but breaks all existing code immediately.

## Consequences

- **Backward compatible**: `WebServicesManager` continues to work, marked `@deprecated`
- **Testable**: `RequestProcessor` can process a single service without a registry
- **Composable**: framework (or any consumer) decides how to find/instantiate services
- **Smaller surface**: each class has one job
- **Migration path**: existing apps keep working, new apps use `RequestProcessor` directly
