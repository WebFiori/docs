# Dependency Injection

The dependency injection (DI) container manages object creation and wiring. It allows you to bind interfaces to implementations, register singletons, and let the framework auto-resolve constructor dependencies.

<meta name="description" content="Learn how to use the WebFiori DI container for binding, singletons, and auto-resolution.">

## Key Concepts

- **Bind** — Map an interface to a concrete class (new instance each time)
- **Singleton** — Map an interface to a class that's instantiated only once
- **Instance** — Register an already-created object
- **Auto-resolution** — The container inspects constructors and resolves typed parameters automatically

## Using the Container

Use `ContainerFacade` for static access:

```php
use WebFiori\Container\ContainerFacade;

// Bind interface to implementation (new instance per make() call)
ContainerFacade::bind(PaymentGatewayInterface::class, StripeGateway::class);

// Bind as singleton (same instance reused)
ContainerFacade::singleton(LoggerInterface::class, FileLogger::class);

// Register an existing instance
ContainerFacade::instance(ConfigInterface::class, $configObject);

// Resolve
$gateway = ContainerFacade::make(PaymentGatewayInterface::class);
```

## Registration

### Transient Binding (New Instance Each Time)

```php
ContainerFacade::bind(NotifierInterface::class, EmailNotifier::class);

$a = ContainerFacade::make(NotifierInterface::class); // new EmailNotifier
$b = ContainerFacade::make(NotifierInterface::class); // new EmailNotifier
// $a !== $b
```

### Singleton Binding (Shared Instance)

```php
ContainerFacade::singleton(CacheInterface::class, RedisCache::class);

$a = ContainerFacade::make(CacheInterface::class); // creates RedisCache
$b = ContainerFacade::make(CacheInterface::class); // returns same instance
// $a === $b
```

### Factory Callable

Pass a callable instead of a class name for custom construction:

```php
ContainerFacade::bind(DatabaseInterface::class, function ($container) {
    $config = $container->make(ConfigInterface::class);
    return new Database($config->get('db.host'), $config->get('db.port'));
});
```

The callable receives the container instance, allowing you to resolve other dependencies.

### Pre-Built Instance

```php
$logger = new FileLogger('/var/log/app');
ContainerFacade::instance(LoggerInterface::class, $logger);
```

## Auto-Resolution

When resolving a class, the container inspects its constructor and automatically resolves type-hinted parameters:

```php
class OrderService {
    public function __construct(
        private PaymentGatewayInterface $gateway,
        private NotifierInterface $notifier
    ) {
    }
}

// If PaymentGatewayInterface and NotifierInterface are bound:
$service = ContainerFacade::make(OrderService::class);
// Both dependencies are injected automatically
```

Resolution rules for constructor parameters:
1. If the type is bound in the container → resolve it
2. If the parameter has a default value → use the default
3. If the type is nullable → pass `null`
4. Otherwise → throw `ContainerException`

## Checking and Removing Bindings

```php
// Check if a binding exists
if (ContainerFacade::has(PaymentGatewayInterface::class)) {
    // ...
}

// Remove a specific binding
ContainerFacade::remove(PaymentGatewayInterface::class);

// Clear all bindings (useful in tests)
ContainerFacade::reset();
```

## Where to Register Bindings

Register bindings during application initialization, typically in `App/Ini/Privileges.php`:

```php
<?php
namespace App\Ini;

use App\Services\PaymentGatewayInterface;
use App\Services\StripeGateway;
use App\Services\MockGateway;
use WebFiori\Container\ContainerFacade;

class Privileges {
    public static function initialize() {
        if (getenv('APP_ENV') === 'testing') {
            ContainerFacade::bind(PaymentGatewayInterface::class, MockGateway::class);
        } else {
            ContainerFacade::bind(PaymentGatewayInterface::class, StripeGateway::class);
        }
    }
}
```

## Using the Container Directly

For testing or when you need multiple containers:

```php
use WebFiori\Container\Container;

$container = new Container();
$container->bind(ServiceInterface::class, ConcreteService::class);
$service = $container->make(ServiceInterface::class);
```

## Best Practices

- Bind interfaces, not concrete classes — makes swapping implementations easy
- Use singletons for stateless services (loggers, caches, gateways)
- Use transient bindings for stateful objects
- Register mocks in test environments for isolation
- Keep bindings in one place (`Privileges.php`) for discoverability
