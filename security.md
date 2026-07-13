# Security & Authorization

WebFiori provides a layered security system combining authentication state management, role-based access control (RBAC), attribute-based access control (ABAC), and declarative annotations for endpoint protection.

<meta name="description" content="Learn how to use WebFiori's security system: SecurityContext, Access, RBAC, ABAC policies, and PreAuthorize.">

## Components Overview

| Component | Package | Role |
|-----------|---------|------|
| `SecurityContext` | `webfiori/http` | Holds current authenticated user for the request |
| `SecurityPrincipal` | `webfiori/http` | Interface for user objects |
| `#[RequiresAuth]` | `webfiori/http` | Annotation: endpoint requires authentication |
| `#[PreAuthorize]` | `webfiori/http` | Annotation: expression-based access control |
| `Access` | `webfiori/framework` | RBAC role/permission definitions |
| `AccessManager` | `webfiori/framework` | RBAC + ABAC policy evaluation |
| `AuthorizeMiddleware` | `webfiori/framework` | Middleware for permission checks |

## SecurityPrincipal

Implement this interface on your user class:

```php
<?php
namespace App\Domain;

use WebFiori\Http\SecurityPrincipal;

class User implements SecurityPrincipal {
    public function __construct(
        public int $id,
        public string $name,
        public string $role
    ) {
    }

    public function getAuthorities(): array {
        return match ($this->role) {
            'admin' => ['users.manage', 'orders.manage', 'orders.view'],
            'customer' => ['orders.create', 'orders.view'],
            default => []
        };
    }

    public function getId(): int|string {
        return $this->id;
    }

    public function getName(): string {
        return $this->name;
    }

    public function getRoles(): array {
        return [$this->role];
    }

    public function isActive(): bool {
        return true;
    }
}
```

## SecurityContext

Set the current user during request processing (typically in middleware):

```php
use WebFiori\Http\SecurityContext;

// Set authenticated user
SecurityContext::setCurrentUser($user);

// Check authentication
SecurityContext::isAuthenticated(); // true if user is set and active

// Get current user
$user = SecurityContext::getCurrentUser();

// Check roles and authorities
SecurityContext::hasRole('admin');
SecurityContext::hasAuthority('orders.manage');

// Clear (e.g., on logout)
SecurityContext::clear();
```

## Declarative Annotations

### `#[RequiresAuth]`

Requires the user to be authenticated. Can be placed on a class (all methods) or individual methods:

```php
use WebFiori\Http\Annotations\RequiresAuth;
use WebFiori\Http\Annotations\RestController;

#[RestController('orders', 'Order API')]
#[RequiresAuth]  // All methods require authentication
class OrderService extends WebService {
    public function isAuthorized(): bool {
        return SecurityContext::isAuthenticated();
    }
    // ...
}
```

### `#[PreAuthorize]`

Evaluates a security expression before the method executes:

```php
use WebFiori\Http\Annotations\PreAuthorize;

// Single role check
#[PreAuthorize("hasRole('admin')")]
public function deleteUser(): array { ... }

// Single authority check
#[PreAuthorize("hasAuthority('orders.manage')")]
public function shipOrder(): array { ... }

// Multiple roles (any)
#[PreAuthorize("hasAnyRole('admin', 'moderator')")]
public function banUser(): array { ... }

// Combined with AND
#[PreAuthorize("isAuthenticated() && hasAuthority('reports.view')")]
public function getReport(): array { ... }

// Combined with OR
#[PreAuthorize("hasRole('admin') || hasAuthority('orders.manage')")]
public function cancelOrder(): array { ... }
```

Supported expressions:

| Expression | Meaning |
|-----------|---------|
| `hasRole('ROLE')` | User has the specified role |
| `hasAnyRole('R1', 'R2')` | User has any of the listed roles |
| `hasAuthority('PERM')` | User has the specified authority/permission |
| `hasAnyAuthority('P1', 'P2')` | User has any of the listed authorities |
| `isAuthenticated()` | User is authenticated and active |
| `permitAll()` | Always allows access |

Combine with `&&` (AND) and `||` (OR).

### `#[AllowAnonymous]`

Skips all authentication checks for a method:

```php
#[AllowAnonymous]
public function getPublicData(): array { ... }
```

## RBAC with Access

Define roles and their permissions during initialization:

```php
use WebFiori\Framework\Access;

// Define roles with permissions
Access::role('customer', ['orders.create', 'orders.view', 'orders.cancel']);
Access::role('staff', ['orders.view', 'orders.update', 'orders.ship']);
Access::role('admin', ['orders.create', 'orders.view', 'orders.cancel',
                       'orders.update', 'orders.ship', 'orders.manage']);

// Assign role to user (for storage-backed scenarios)
Access::assignRoleToUser($userId, 'customer');
```

Permission and group IDs can contain letters (`A-Z`, `a-z`), digits (`0-9`), underscores (`_`), dots (`.`), and dashes (`-`). Dots and dashes are useful as namespace separators (e.g., `orders.create`, `billing-read`).

### Role Inheritance

Roles can inherit permissions from a parent role:

```php
Access::role('manager', ['reports.view'])->inherits('staff');
// manager now has: reports.view + all staff permissions
```

### Checking Permissions

```php
// Pass user object — reads roles from getRoles() if no internal mapping exists
Access::can($user, 'orders.cancel');

// Pass user ID — uses roles from assignRoleToUser() or storage
Access::can(42, 'orders.cancel');
```

### Storage Backends

By default, roles are stored in memory. For persistence across requests, use a storage backend:

| Backend | Class | Use Case |
|---------|-------|----------|
| In-memory | `InMemoryAccessStorage` | Testing, stateless APIs |
| File-based | `FileAccessStorage` | Simple deployments |
| Database | `DatabaseAccessStorage` | Production with DB |

```php
use WebFiori\Framework\AccessManager;
use WebFiori\Framework\Storage\DatabaseAccessStorage;

$manager = new AccessManager(new DatabaseAccessStorage($connectionName));
$manager->loadFromStorage(); // load roles/permissions from DB
```

To persist changes:

```php
$manager->saveToStorage();
```

## ABAC with Policies

Policies add resource-level rules on top of RBAC. A policy is evaluated only after the RBAC check passes.

### Creating a Policy

Create policy classes under `App/Policies/`:

```php
<?php
namespace App\Policies;

use App\Domain\Order;

class OrderCancelPolicy {
    public function getPermission(): string {
        return 'orders.cancel';
    }

    public function evaluate($user, ?object $resource = null): bool {
        if ($resource === null || !$resource instanceof Order) {
            return false;
        }

        // Only pending orders can be cancelled
        if ($resource->status !== 'pending') {
            return false;
        }

        // Admin can cancel any pending order
        if (in_array('admin', $user->getRoles())) {
            return true;
        }

        // Customers can only cancel their own orders
        return $user->getId() === $resource->userId;
    }
}
```

### Registering Policies

```php
use App\Policies\OrderCancelPolicy;
use WebFiori\Framework\Access;

Access::registerPolicy(new OrderCancelPolicy());
```

### Using Policies

Pass the resource as the third argument to `can()`:

```php
$order = $orderRepo->findById($id);

if (!Access::can($user, 'orders.cancel', $order)) {
    throw new ForbiddenException('You cannot cancel this order.');
}
```

The evaluation flow:
1. **RBAC check**: Does the user's role have the `orders.cancel` permission? If no → denied.
2. **Policy check**: Does `OrderCancelPolicy::evaluate($user, $order)` return true? If no → denied.
3. Both pass → allowed.

## Putting It Together

A typical setup in `App/Ini/Privileges.php`:

```php
<?php
namespace App\Ini;

use App\Policies\OrderCancelPolicy;
use App\Policies\OrderViewPolicy;
use WebFiori\Framework\Access;

class Privileges {
    public static function initialize() {
        // RBAC
        Access::role('customer', ['orders.create', 'orders.view', 'orders.cancel']);
        Access::role('admin', ['orders.create', 'orders.view', 'orders.cancel', 'orders.manage']);

        // ABAC
        Access::registerPolicy(new OrderViewPolicy());
        Access::registerPolicy(new OrderCancelPolicy());
    }
}
```

A middleware that loads the user into SecurityContext:

```php
<?php
namespace App\Middleware;

use WebFiori\Framework\Access;
use WebFiori\Framework\Middleware\AbstractMiddleware;
use WebFiori\Http\SecurityContext;

class SecurityContextLoader extends AbstractMiddleware {
    public function before(Request $request, Response $response) {
        $userId = SessionsManager::get('user-id');

        if ($userId === null) {
            SecurityContext::clear();
            return;
        }

        $user = $this->loadUser($userId);

        if ($user !== null && $user->isActive()) {
            SecurityContext::setCurrentUser($user);
            Access::assignRoleToUser($user->getId(), $user->role);
        }
    }
}
```

A service using both annotations and `Access::can()`:

```php
#[RestController('orders', 'Order API')]
#[RequiresAuth]
class OrderService extends WebService {
    public function isAuthorized(): bool {
        return SecurityContext::isAuthenticated();
    }

    #[DeleteMapping]
    #[PreAuthorize("hasAuthority('orders.cancel')")]
    public function cancelOrder(?int $id = null): array {
        $order = $this->repo->findById($id);
        $user = SecurityContext::getCurrentUser();

        // ABAC: policy checks ownership + status
        if (!Access::can($user, 'orders.cancel', $order)) {
            throw new ForbiddenException('Cannot cancel this order.');
        }

        $order->status = 'cancelled';
        $this->repo->save($order);
        return [$order];
    }
}
```
