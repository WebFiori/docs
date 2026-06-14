# Built-in Middleware

WebFiori ships with several middleware for common security and performance concerns. They can be assigned to routes individually or via middleware groups.

<meta name="description" content="Reference for WebFiori's built-in middleware: rate limiting, CORS, CSRF, maintenance mode, and HTTP caching.">

## Rate Limiting

Limits requests per time window per client IP. Returns `429 Too Many Requests` when exceeded.

```php
use WebFiori\Framework\Middleware\RateLimitMiddleware;

new RateLimitMiddleware(
    maxRequests: 60,      // requests allowed per window
    windowSeconds: 60,    // window duration
    keyResolver: null,    // custom key function (optional)
    trustedIps: []        // IPs that bypass limiting (optional)
)
```

Response headers added:
- `X-RateLimit-Limit` — max requests
- `X-RateLimit-Remaining` — requests left in window
- `X-RateLimit-Reset` — Unix timestamp when window resets
- `Retry-After` — seconds to wait (only on 429)

Usage:

```php
Router::api([
    RouteOption::PATH => '/apis/{service}',
    RouteOption::TO => MyManager::class,
    RouteOption::MIDDLEWARE => [
        new RateLimitMiddleware(maxRequests: 100, windowSeconds: 60)
    ]
]);
```

## CORS (Cross-Origin Resource Sharing)

Handles preflight OPTIONS requests and adds CORS headers.

```php
use WebFiori\Framework\Middleware\CorsMiddleware;

new CorsMiddleware([
    'origins' => ['https://app.example.com'],  // allowed origins (* for all)
    'methods' => ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
    'headers' => ['Content-Type', 'Authorization', 'X-CSRF-TOKEN'],
    'max-age' => 86400,        // preflight cache duration (seconds)
    'credentials' => true,     // allow cookies/auth headers
    'expose-headers' => [],    // headers visible to client JS
])
```

## CSRF Protection

Validates a CSRF token on state-changing requests (POST, PUT, DELETE, PATCH). Token is checked from the `X-CSRF-TOKEN` header or `_csrf` request parameter.

```php
use WebFiori\Framework\Middleware\VerifyCsrfToken;
```

Registered name: `csrf`. Belongs to the `web` middleware group.

For server-rendered pages, the middleware auto-injects a `<meta name="csrf-token">` tag in the response head. JavaScript can read it:

```javascript
const token = document.querySelector('meta[name="csrf-token"]').content;
fetch('/api/endpoint', {
    method: 'POST',
    headers: { 'X-CSRF-TOKEN': token }
});
```

## Maintenance Mode

Returns `503 Service Unavailable` when the application is in maintenance mode.

```php
use WebFiori\Framework\Middleware\CheckMaintenanceMode;
```

Registered name: `maintenance-check`. Belongs to both `web` and `api` groups.

Activate maintenance mode by creating `App/Storage/.maintenance`:

```json
{
    "message": "We are upgrading the system. Back in 30 minutes.",
    "retry_after": 1800,
    "allowed": ["192.168.1.100"],
    "api_prefix": "/apis"
}
```

- `allowed` — IPs that bypass maintenance mode
- `retry_after` — value for the `Retry-After` header
- `api_prefix` — path prefix to detect API requests (returns JSON instead of HTML)

Remove the file to exit maintenance mode.

## HTTP Caching (ETag / 304)

Adds ETag and Cache-Control headers to GET responses. Returns `304 Not Modified` when the client's `If-None-Match` header matches.

```php
use WebFiori\Framework\Middleware\HttpCacheMiddleware;

new HttpCacheMiddleware([
    'max-age' => 300,    // Cache-Control max-age in seconds (0 = no header)
    'public' => true,    // Cache-Control: public (shared caches like CDNs)
])
```

Best for read-heavy, infrequently-changing endpoints (product catalogs, static configs).

## Session Start

Starts the session. Required by middleware that reads session data (CSRF, auth, rate limiter with session keys).

Registered name: `start-session`.

## Authorize

Checks if the current user has a specific permission via `Access::can()`.

```php
use WebFiori\Framework\Middleware\AuthorizeMiddleware;

new AuthorizeMiddleware('orders.manage')
```

Returns 401 if no user in session, 403 if user lacks the permission.

## Assigning Middleware to Routes

### Individual assignment:

```php
Router::api([
    RouteOption::PATH => '/apis/{service}',
    RouteOption::TO => MyManager::class,
    RouteOption::MIDDLEWARE => [
        'start-session',
        'csrf',
        new RateLimitMiddleware(maxRequests: 60, windowSeconds: 60),
        new CorsMiddleware(['origins' => ['*']]),
    ]
]);
```

### By group name:

Middleware can belong to groups (set via `addToGroup()` in the middleware constructor). Assign a group name to apply all middleware in that group:

```php
RouteOption::MIDDLEWARE => ['web']  // applies all middleware in the 'web' group
```

## Middleware Dependencies

Middleware can declare dependencies via `getDependencies()`. When a middleware is assigned to a route, the framework automatically pulls in all transitive dependencies from the middleware registry and sorts them using topological sort (Kahn's algorithm). Priority is used as a tiebreaker for unrelated middleware.

```php
class MyMiddleware extends AbstractMiddleware {
    public function getDependencies(): array {
        return ['start-session']; // start-session is auto-included and runs first
    }
}
```

You only need to assign the "leaf" middleware to a route — its entire dependency chain is resolved automatically:

```php
// Only assign MyMiddleware — start-session is pulled in from the registry
RouteOption::MIDDLEWARE => ['my-middleware']
```

If a declared dependency is not registered in `MiddlewareManager`, it is silently skipped. Circular dependencies throw a `RoutingException`.
