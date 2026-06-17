# Event Dispatcher

The event dispatcher enables decoupled communication between components. Instead of calling services directly, you dispatch an event and let registered listeners react to it independently.

<meta name="description" content="Learn how to use the WebFiori event dispatcher for decoupled, event-driven architecture.">

## Key Concepts

- **Event** — A plain PHP object representing something that happened
- **Listener** — A callable or object that reacts to an event
- **Dispatcher** — Routes events to their registered listeners

## Creating an Event

An event is any PHP class. Use public readonly properties or getters to carry data:

```php
<?php
namespace App\Events;

use App\Domain\Order;

class OrderPlacedEvent {
    public function __construct(
        public readonly Order $order,
        public readonly array $items
    ) {
    }
}
```

## Registering Listeners

Register listeners in `App/Ini/Privileges.php` (or any initialization file) using `EventDispatcherFacade`:

```php
use App\Events\OrderPlacedEvent;
use App\Listeners\SendConfirmationListener;
use App\Listeners\UpdateInventoryListener;
use WebFiori\Event\EventDispatcherFacade;

// Class-based listener (must have a handle() method)
EventDispatcherFacade::listen(OrderPlacedEvent::class, new SendConfirmationListener());
EventDispatcherFacade::listen(OrderPlacedEvent::class, new UpdateInventoryListener());

// Callable listener
EventDispatcherFacade::listen(OrderPlacedEvent::class, function (OrderPlacedEvent $event) {
    // inline handling
});
```

Multiple listeners can be registered for the same event. They execute in registration order.

## Creating a Listener

A class-based listener must have a `handle()` method that accepts the event:

```php
<?php
namespace App\Listeners;

use App\Events\OrderPlacedEvent;
use WebFiori\Queue\QueueFacade;
use App\Jobs\SendOrderEmailJob;

class SendConfirmationListener {
    public function handle(OrderPlacedEvent $event): void {
        QueueFacade::dispatch(new SendOrderEmailJob($event->order->id));
    }
}
```

## Dispatching Events

Dispatch an event from anywhere in your application:

```php
use App\Events\OrderPlacedEvent;
use WebFiori\Event\EventDispatcherFacade;

$event = new OrderPlacedEvent($order, $items);
EventDispatcherFacade::dispatch($event);
```

All registered listeners for `OrderPlacedEvent` will be called synchronously in registration order.

## Built-in Framework Events

The framework dispatches these events automatically:

| Event | When |
|-------|------|
| `WebFiori\Framework\Events\RequestReceived` | After a request is received and before routing |
| `WebFiori\Framework\Events\ResponseSent` | After the response is sent to the client |
| `WebFiori\Framework\Events\RouteNotFound` | When no route matches the request |
| `WebFiori\Framework\Events\SessionStarted` | When a session is started |
| `WebFiori\Framework\Events\SessionDestroyed` | When a session is destroyed |

You can listen to these for logging, metrics, or custom behavior:

```php
use WebFiori\Framework\Events\RequestReceived;
use WebFiori\Event\EventDispatcherFacade;

EventDispatcherFacade::listen(RequestReceived::class, function (RequestReceived $event) {
    // Log every incoming request
});
```

## Using the Dispatcher Directly

For dependency injection or testing, use `EventDispatcher` directly instead of the facade:

```php
use WebFiori\Event\EventDispatcher;

$dispatcher = new EventDispatcher();
$dispatcher->listen(OrderPlacedEvent::class, $listener);
$dispatcher->dispatch($event);

// Check registered listeners
$count = $dispatcher->getListenerCount();
$listeners = $dispatcher->getListeners(OrderPlacedEvent::class);
```

## Best Practices

- Keep events immutable (use `readonly` properties)
- Name events in past tense (`OrderPlaced`, `PaymentFailed`, `UserRegistered`)
- Keep listeners focused — one listener, one side effect
- Use the job queue inside listeners for heavy work (keeps dispatch fast)
- Don't rely on listener execution order for correctness

## Using as a Command Bus

The event dispatcher can also serve as a lightweight command bus. The difference is semantic: events describe something that happened (past tense), commands describe an action to perform (imperative).

### Defining a Command

A command is a plain PHP class representing an intent:

```php
<?php
namespace App\Commands;

class UpdateIndicatorStatus {
    public function __construct(
        public readonly array $indicatorIds,
        public readonly string $newStatus,
        public readonly int $userId
    ) {
    }
}
```

### Defining a Handler

A handler processes exactly one command:

```php
<?php
namespace App\Handlers;

use App\Commands\UpdateIndicatorStatus;

class UpdateIndicatorStatusHandler {
    public function handle(UpdateIndicatorStatus $cmd): void {
        // validate transition
        // update database
        // dispatch domain event
    }
}
```

### Registering and Dispatching

```php
use App\Commands\UpdateIndicatorStatus;
use App\Handlers\UpdateIndicatorStatusHandler;
use WebFiori\Event\EventDispatcherFacade;

// Register (one handler per command)
EventDispatcherFacade::listen(UpdateIndicatorStatus::class, new UpdateIndicatorStatusHandler());

// Dispatch from a controller
EventDispatcherFacade::dispatch(new UpdateIndicatorStatus(
    indicatorIds: [1, 2, 3],
    newStatus: 'S01',
    userId: 42
));
```

### When to Use Commands vs Events

| Use Commands | Use Events |
|---|---|
| "Do this" — imperative action | "This happened" — notification |
| Exactly one handler | Zero or many listeners |
| Controller → Handler (replaces inline logic) | Handler → Listeners (side effects) |
| `CreateUser`, `SendForm`, `ApproveIndicator` | `UserCreated`, `FormSent`, `IndicatorApproved` |

A typical flow combines both:

```
Controller → dispatches Command → Handler executes logic → dispatches Event → Listeners react
```

### Convention

To keep the codebase clear, use separate directories:

```
App/Commands/          — command classes (imperative: verb + noun)
App/Handlers/          — command handlers (one per command)
App/Events/            — event classes (past tense: noun + verb)
App/Listeners/         — event listeners (multiple per event)
```
