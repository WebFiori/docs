# ADR-0006: ServiceRouter — Framework Integration with RequestProcessor

**Date:** 2026-06-01
**Status:** Accepted

## Context

After ADR-0005 introduces `RequestProcessor` in `webfiori/http`, the framework needs a way to route requests directly to `WebService` classes without requiring a `WebServicesManager`.

The goal: developer creates a controller with `#[RestController]`, and the framework handles discovery, routing, and processing automatically.

## Decision

Introduce `ServiceRouter` in `webfiori/framework` with two modes of operation:

### Mode 1: Auto-Discover at Boot (Recommended)

Scans a namespace, finds all `#[RestController]` classes, and registers a real route per service in the Router at boot time.

```php
// App/Ini/Routes/APIsRoutes.php
class APIsRoutes {
    public static function create() {
        ServiceRouter::discover('App\\Apis', '/apis', [
            RouteOption::MIDDLEWARE => ['start-session', 'security-context',
                new CorsMiddleware(), new RateLimitMiddleware(60, 60)]
        ]);
    }
}
```

Produces individual routes:
- `GET|POST|PUT|DELETE /apis/orders` → `OrderService`
- `GET|POST /apis/products` → `ProductService`
- `POST /apis/auth` → `AuthService`

Each route is a real entry in the Router — visible in `routes:list`, individually configurable.

### Mode 2: Dynamic Resolution at Runtime (Flexible)

A single catch-all route that resolves the controller name from the URL at request time.

```php
Router::api([
    RouteOption::PATH => '/apis/{controller}',
    RouteOption::NAMESPACE => 'App\\Apis',
    RouteOption::MIDDLEWARE => [...]
]);
```

The framework resolves `{controller}` → class with matching `#[RestController]` name at request time.

### When to Use Which

| Criteria | Auto-discover (Mode 1) | Dynamic (Mode 2) |
|----------|----------------------|------------------|
| Performance | Routes resolved at boot | Resolution per request |
| Visibility | Each service in `routes:list` | Single catch-all entry |
| Per-service middleware | Supported (override per service) | Same for all services |
| Plugin/module systems | Requires restart on new service | Drop-in — add file, it's live |
| Development workflow | Cache-clear on new service | Instant |

Both modes use `RequestProcessor` internally.

## ServiceRouter API

```php
class ServiceRouter {
    /**
     * Mode 1: Discover services in namespace and register routes.
     *
     * @param string $namespace  Namespace to scan (e.g., 'App\\Apis')
     * @param string $basePath   URL prefix (e.g., '/apis')
     * @param array  $options    Shared route options (middleware, etc.)
     */
    public static function discover(string $namespace, string $basePath, array $options = []): void;

    /**
     * Mode 2: Handle a dynamic controller request.
     * Called by the router when RouteOption::NAMESPACE is set.
     *
     * @param string   $controllerName  The {controller} path parameter value
     * @param string   $namespace       Namespace to search
     * @param Request  $request
     * @param Response $response
     */
    public static function handle(string $controllerName, string $namespace, Request $request, Response $response): void;
}
```

## Discovery Logic

```php
private static function scanNamespace(string $namespace): array {
    $map = []; // name → ['class' => FQCN, 'type' => 'service'|'manager']

    foreach (ClassLoader::getClassesInNamespace($namespace) as $class) {
        $ref = new ReflectionClass($class);

        // Priority 1: #[RestController] attribute
        $attrs = $ref->getAttributes(RestController::class);
        if (!empty($attrs)) {
            $attr = $attrs[0]->newInstance();
            $name = !empty($attr->name) ? $attr->name : self::deriveNameFromClass($ref->getShortName());
            $map[$name] = ['class' => $class, 'type' => 'service'];
            continue;
        }

        // Priority 2: WebServicesManager subclass (traditional manager)
        if ($ref->isSubclassOf(WebServicesManager::class) && !$ref->isAbstract()) {
            $name = self::deriveNameFromClass($ref->getShortName());
            $map[$name] = ['class' => $class, 'type' => 'manager'];
            continue;
        }

        // Priority 3: WebService subclass without attribute
        if ($ref->isSubclassOf(WebService::class) && !$ref->isAbstract()) {
            $instance = new $class();
            $name = $instance->getName();
            $map[$name] = ['class' => $class, 'type' => 'service'];
        }
    }

    return $map;
}
```

The discovery order ensures that attributed classes are preferred. Non-attributed `WebService` subclasses use their `getName()` value. `WebServicesManager` subclasses are registered as traditional manager routes (the manager handles its own sub-service dispatch internally).

## Instantiation

Services are instantiated via the DI container when available:

```php
$service = ContainerFacade::has($class)
    ? ContainerFacade::make($class)
    : new $class();
```

This enables constructor injection for services.

## Implementation Steps

| # | Task | Depends on |
|---|------|-----------|
| 1 | Add `ServiceRouter::discover()` — scan namespace, register routes | ADR-0005 Step 4 (RequestProcessor) |
| 2 | Add `RouteOption::NAMESPACE` support in Router for dynamic mode | ADR-0005 Step 4 |
| 3 | Integrate with `ContainerFacade` for service instantiation | — |
| 4 | Add `services:list` CLI command | Step 1 |
| 5 | Add production caching for discovery (via `CacheFacade`) | Step 1 |

## Backward Compatibility

- `RouteOption::TO => MyManager::class` continues to work (unchanged)
- `ServiceRouter::discover()` and `RouteOption::NAMESPACE` are new, opt-in features
- Both old and new styles can coexist in the same application
- `WebServicesManager` (deprecated in ADR-0005) remains functional

## Consequences

- **Zero boilerplate**: create a `#[RestController]` class → it's routable
- **Explicit**: namespace and base path must be configured (no magic global scanning)
- **Debuggable**: Mode 1 routes appear in `routes:list`
- **Flexible**: Mode 2 supports plugin architectures and rapid development
- **DI-ready**: services get constructor dependencies injected
- **Testable**: `RequestProcessor` works directly in tests without routing

## GitHub Issues

| Step | Issue |
|------|-------|
| 1 | [webfiori/framework#382](https://github.com/WebFiori/framework/issues/382) — ServiceRouter::discover() |
| 2 | [webfiori/framework#383](https://github.com/WebFiori/framework/issues/383) — RouteOption::NAMESPACE |
| 3 | [webfiori/framework#384](https://github.com/WebFiori/framework/issues/384) — DI container integration |
| 4 | [webfiori/framework#385](https://github.com/WebFiori/framework/issues/385) — services:list CLI command |
| 5 | [webfiori/framework#386](https://github.com/WebFiori/framework/issues/386) — Production caching |

**Prerequisite:** [webfiori/http#121](https://github.com/WebFiori/http/issues/121) (RequestProcessor) from ADR-0005.
