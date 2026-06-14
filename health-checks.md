# Health Checks

Health checks verify that your application's dependencies (database, cache, external services) are operational. The framework provides a registry, built-in checks, and an HTTP endpoint for monitoring.

<meta name="description" content="Learn how to use WebFiori health checks for monitoring application dependencies.">

## Registering Checks

Register checks during initialization (e.g., in `App/Ini/Privileges.php`):

```php
use WebFiori\Framework\Health\HealthCheck;
use WebFiori\Framework\Health\Checks\CacheCheck;
use App\Health\DatabaseCheck;

HealthCheck::register(new DatabaseCheck());
HealthCheck::register(new CacheCheck()); // built-in
```

## Creating a Custom Check

Implement `HealthCheckInterface`:

```php
<?php
namespace App\Health;

use WebFiori\Framework\App;
use WebFiori\Framework\Health\HealthCheckInterface;
use WebFiori\Framework\Health\HealthCheckResult;

class DatabaseCheck implements HealthCheckInterface {
    public function getName(): string {
        return 'database';
    }

    public function check(): HealthCheckResult {
        try {
            $conn = App::getConfig()->getDBConnection('main');
            $db = new \WebFiori\Database\Database($conn);
            $db->raw("SELECT 1")->execute();

            return HealthCheckResult::ok();
        } catch (\Throwable $e) {
            return HealthCheckResult::fail($e->getMessage());
        }
    }
}
```

You can also register a callable:

```php
HealthCheck::register('redis', function () {
    // return HealthCheckResult::ok() or HealthCheckResult::fail('reason')
});
```

## Running Checks

```php
use WebFiori\Framework\Health\HealthCheck;

$result = HealthCheck::runAll();
// Returns:
// [
//     'status' => 'ok' | 'fail',
//     'timestamp' => '2026-05-31T12:00:00+00:00',
//     'checks' => [
//         'database' => ['status' => 'ok'],
//         'cache' => ['status' => 'ok'],
//     ]
// ]
```

## HTTP Endpoint

Expose health checks as an API endpoint:

```php
<?php
namespace App\Apis;

use WebFiori\Framework\Health\HealthCheck;
use WebFiori\Http\Annotations\AllowAnonymous;
use WebFiori\Http\Annotations\GetMapping;
use WebFiori\Http\Annotations\ResponseBody;
use WebFiori\Http\Annotations\RestController;
use WebFiori\Http\WebService;

#[RestController('health', 'Health check API')]
class HealthService extends WebService {
    #[GetMapping]
    #[ResponseBody]
    #[AllowAnonymous]
    public function check(): array {
        $result = HealthCheck::runAll();
        $this->getManager()->getResponse()->setCode(
            $result['status'] === 'ok' ? 200 : 503
        );

        return $result;
    }
}
```

Returns `200` when all checks pass, `503` when any check fails. Suitable for load balancer health probes.

## Built-in Checks

| Check | What it verifies |
|-------|-----------------|
| `CacheCheck` | Cache write/read/delete cycle works |

## Introspection

Retrieve all registered checks for dashboards or debugging:

```php
$checks = HealthCheck::getChecks();
// Returns associative array keyed by check name.
// Values are HealthCheckInterface instances or callables.

$count = HealthCheck::getCheckCount();
```

## Post-Run Hooks

Register callbacks that execute after all checks complete. Useful for notifications:

```php
HealthCheck::afterAll(function (array $results) {
    if ($results['status'] === 'fail') {
        // Send alert to Slack, email, etc.
        NotificationService::alert('Health check failed', $results);
    }
});

// Callbacks receive the same aggregate array returned by runAll()
$result = HealthCheck::runAll(); // afterAll callbacks fire here
```

Multiple callbacks can be registered — they execute in order. Call `HealthCheck::reset()` to clear both checks and callbacks.

## Best Practices

- Keep checks fast (< 1 second each) — they run on every probe
- Return `503` on failure so load balancers can route traffic away
- Don't expose sensitive details in failure messages for public endpoints
- Check all critical dependencies: database, cache, queue, external APIs
