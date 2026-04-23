# MVC Architecture

<meta name="description" content="Learn how to implement MVC architecture in WebFiori Framework using API Controllers, Repositories, and Entities.">

In this page:
* [Introduction](#introduction)
* [Architecture Overview](#architecture-overview)
* [Creating Entities](#creating-entities)
* [Creating Repositories](#creating-repositories)
  * [Basic Repository](#basic-repository)
  * [Custom Query Methods](#custom-query-methods)
  * [Pagination](#pagination)
* [Creating API Controllers](#creating-api-controllers)
  * [Using Annotations](#using-annotations)
  * [Traditional Approach](#traditional-approach)
  * [Request Parameters](#request-parameters)
  * [Entity Mapping](#entity-mapping)
* [Putting It All Together](#putting-it-all-together)
* [Folder Structure](#folder-structure)
* [Best Practices](#best-practices)

## Introduction

WebFiori Framework supports the Model-View-Controller (MVC) architectural pattern for building well-structured APIs. The framework provides:

- **Entities (Model)**: Plain PHP classes representing your domain objects
- **Repositories**: Data access layer using `AbstractRepository` for database operations
- **API Controllers (Controller)**: Web services that handle HTTP requests using `WebService` class

This separation of concerns makes your code more maintainable, testable, and scalable.

## Architecture Overview

The recommended folder structure follows clean architecture principles:

| Layer | Location | Purpose |
|-------|----------|---------|
| Domain | `App/Domain` | Pure entity classes with no framework dependencies |
| Infrastructure | `App/Infrastructure/Repository` | Repository implementations |
| Infrastructure | `App/Infrastructure/Schema` | Database table definitions |
| API | `App/Apis` | API controllers (web services) |

## Creating Entities

Entities are simple PHP classes that represent your domain objects. They should be framework-agnostic.

``` php
namespace App\Domain;

class User {
    public function __construct(
        public ?int $id = null,
        public string $name = '',
        public string $email = '',
        public int $age = 0
    ) {
    }
}
```

For more complex entities with validation:

``` php
namespace App\Domain;

class Product {
    public ?int $id = null;
    public string $name;
    public string $category;
    public float $price;
    public int $stock;

    public function __construct(
        string $name = '', 
        string $category = '', 
        float $price = 0, 
        int $stock = 0
    ) {
        $this->name = $name;
        $this->category = $category;
        $this->price = $price;
        $this->stock = $stock;
    }

    public function isInStock(): bool {
        return $this->stock > 0;
    }

    public function toArray(): array {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'category' => $this->category,
            'price' => $this->price,
            'stock' => $this->stock
        ];
    }
}
```

## Creating Repositories

Repositories handle all database operations for your entities. Extend `AbstractRepository` to get built-in CRUD operations.

### Basic Repository

``` php
namespace App\Infrastructure\Repository;

use App\Domain\User;
use WebFiori\Database\Repository\AbstractRepository;

class UserRepository extends AbstractRepository {
    protected function getTableName(): string {
        return 'users';
    }

    protected function getIdField(): string {
        return 'id';
    }

    protected function toArray(object $entity): array {
        return [
            'id' => $entity->id,
            'name' => $entity->name,
            'email' => $entity->email,
            'age' => $entity->age
        ];
    }

    protected function toEntity(array $row): User {
        return new User(
            (int) $row['id'],
            $row['name'],
            $row['email'],
            (int) $row['age']
        );
    }
}
```

The `AbstractRepository` provides these methods out of the box:

| Method | Description |
|--------|-------------|
| `save($entity)` | Insert or update an entity |
| `findById($id)` | Find entity by primary key |
| `findAll()` | Get all entities |
| `deleteById($id)` | Delete by primary key |
| `count()` | Count total records |
| `paginate($page, $perPage)` | Offset-based pagination |
| `paginateByCursor($cursor, $limit)` | Cursor-based pagination |

### Custom Query Methods

Add custom methods for specific queries:

``` php
class UserRepository extends AbstractRepository {
    // ... base methods ...

    public function findByEmail(string $email): ?User {
        $result = $this->getDatabase()->table($this->getTableName())
            ->select()
            ->where('email', $email)
            ->execute();

        if ($result->getCount() === 0) {
            return null;
        }

        return $this->toEntity($result->fetch());
    }

    public function findByAgeRange(int $minAge, int $maxAge): array {
        $result = $this->getDatabase()->table($this->getTableName())
            ->select()
            ->where('age', $minAge, '>=')
            ->andWhere('age', $maxAge, '<=')
            ->execute();

        return array_map(fn($row) => $this->toEntity($row), $result->fetchAll());
    }

    public function findActive(): array {
        $result = $this->getDatabase()->table($this->getTableName())
            ->select()
            ->where('status', 'active')
            ->orderBy(['created_at' => 'DESC'])
            ->execute();

        return array_map(fn($row) => $this->toEntity($row), $result->fetchAll());
    }
}
```

### Pagination

The repository supports both offset-based and cursor-based pagination:

``` php
// Offset-based pagination
$page = $userRepo->paginate(page: 1, perPage: 20);

echo "Total: " . $page->getTotalItems();
echo "Pages: " . $page->getTotalPages();

foreach ($page->getItems() as $user) {
    echo $user->name;
}

// Cursor-based pagination (better for large datasets)
$page = $userRepo->paginateByCursor(cursor: null, limit: 20);

foreach ($page->getItems() as $user) {
    echo $user->name;
}

// Get next page
$nextPage = $userRepo->paginateByCursor(cursor: $page->getNextCursor(), limit: 20);
```

## Creating API Controllers

API controllers handle HTTP requests and return responses. WebFiori supports both annotation-based and traditional approaches.

### Using Annotations

The recommended approach uses PHP 8 attributes for clean, declarative API definitions:

``` php
namespace App\Apis;

use App\Domain\User;
use App\Infrastructure\Repository\UserRepository;
use WebFiori\Http\Annotations\AllowAnonymous;
use WebFiori\Http\Annotations\DeleteMapping;
use WebFiori\Http\Annotations\GetMapping;
use WebFiori\Http\Annotations\MapEntity;
use WebFiori\Http\Annotations\PostMapping;
use WebFiori\Http\Annotations\PutMapping;
use WebFiori\Http\Annotations\RequestParam;
use WebFiori\Http\Annotations\ResponseBody;
use WebFiori\Http\Annotations\RestController;
use WebFiori\Http\WebService;

#[RestController('users', 'User management API')]
class UserController extends WebService {
    private UserRepository $userRepo;

    public function __construct() {
        parent::__construct();
        // Initialize repository with database connection
        $this->userRepo = new UserRepository(App::getDatabase());
    }

    #[GetMapping]
    #[ResponseBody]
    #[AllowAnonymous]
    public function getAll(): array {
        $users = $this->userRepo->findAll();
        return [
            'users' => array_map(fn($u) => $u->toArray(), $users)
        ];
    }

    #[GetMapping]
    #[ResponseBody]
    #[AllowAnonymous]
    #[RequestParam(name: 'id', type: 'int')]
    public function getById(): array {
        $id = $this->getParamVal('id');
        $user = $this->userRepo->findById($id);

        if ($user === null) {
            $this->sendResponse('User not found', self::E, 404);
            return [];
        }

        return ['user' => $user->toArray()];
    }

    #[PostMapping]
    #[ResponseBody]
    #[MapEntity(User::class)]
    public function create(User $user): array {
        $this->userRepo->save($user);
        return [
            'message' => 'User created successfully',
            'user' => $user->toArray()
        ];
    }

    #[PutMapping]
    #[ResponseBody]
    #[RequestParam(name: 'id', type: 'int')]
    #[RequestParam(name: 'name', type: 'string', optional: true)]
    #[RequestParam(name: 'email', type: 'string', optional: true)]
    public function update(): array {
        $id = $this->getParamVal('id');
        $user = $this->userRepo->findById($id);

        if ($user === null) {
            $this->sendResponse('User not found', self::E, 404);
            return [];
        }

        if ($this->getParamVal('name')) {
            $user->name = $this->getParamVal('name');
        }
        if ($this->getParamVal('email')) {
            $user->email = $this->getParamVal('email');
        }

        $this->userRepo->save($user);
        return ['user' => $user->toArray()];
    }

    #[DeleteMapping]
    #[ResponseBody]
    #[RequestParam(name: 'id', type: 'int')]
    public function delete(): array {
        $id = $this->getParamVal('id');
        $this->userRepo->deleteById($id);
        return ['message' => 'User deleted successfully'];
    }
}
```

### Traditional Approach

For more control, use the traditional approach by extending `AbstractWebService`:

``` php
namespace App\Apis;

use App\Infrastructure\Repository\UserRepository;
use WebFiori\Http\AbstractWebService;
use WebFiori\Http\ParamOption;
use WebFiori\Http\ParamType;
use WebFiori\Http\RequestMethod;

class GetUsersService extends AbstractWebService {
    private UserRepository $userRepo;

    public function __construct() {
        parent::__construct('get-users');
        $this->setDescription('Get all users or a specific user by ID');
        $this->setRequestMethods([RequestMethod::GET]);
        
        $this->addParameters([
            'id' => [
                ParamOption::TYPE => ParamType::INT,
                ParamOption::OPTIONAL => true,
                ParamOption::DESCRIPTION => 'User ID to fetch'
            ],
            'page' => [
                ParamOption::TYPE => ParamType::INT,
                ParamOption::OPTIONAL => true,
                ParamOption::DEFAULT => 1
            ],
            'per-page' => [
                ParamOption::TYPE => ParamType::INT,
                ParamOption::OPTIONAL => true,
                ParamOption::DEFAULT => 20
            ]
        ]);

        $this->userRepo = new UserRepository(App::getDatabase());
    }

    public function isAuthorized(): bool {
        return true; // Add your authorization logic
    }

    public function processRequest() {
        $id = $this->getParamVal('id');

        if ($id !== null) {
            $user = $this->userRepo->findById($id);
            
            if ($user === null) {
                $this->sendResponse('User not found', self::E, 404);
                return;
            }
            
            $this->send('application/json', ['user' => $user->toArray()]);
            return;
        }

        $page = $this->getParamVal('page');
        $perPage = $this->getParamVal('per-page');
        $result = $this->userRepo->paginate($page, $perPage);

        $this->send('application/json', [
            'users' => array_map(fn($u) => $u->toArray(), $result->getItems()),
            'pagination' => [
                'page' => $result->getPage(),
                'per_page' => $result->getPerPage(),
                'total' => $result->getTotalItems(),
                'total_pages' => $result->getTotalPages()
            ]
        ]);
    }
}
```

### Request Parameters

Define parameters with validation:

``` php
$this->addParameters([
    'email' => [
        ParamOption::TYPE => ParamType::EMAIL,
        ParamOption::DESCRIPTION => 'User email address'
    ],
    'age' => [
        ParamOption::TYPE => ParamType::INT,
        ParamOption::MIN => 0,
        ParamOption::MAX => 150
    ],
    'status' => [
        ParamOption::TYPE => ParamType::STRING,
        ParamOption::OPTIONAL => true,
        ParamOption::DEFAULT => 'active'
    ],
    'tags' => [
        ParamOption::TYPE => ParamType::ARR,
        ParamOption::OPTIONAL => true
    ]
]);
```

Or using annotations:

``` php
#[RequestParam(name: 'email', type: 'email')]
#[RequestParam(name: 'age', type: 'int')]
#[RequestParam(name: 'status', type: 'string', optional: true, default: 'active')]
public function createUser(): array {
    // ...
}
```

### Entity Mapping

The `#[MapEntity]` attribute automatically maps request parameters to entity properties:

``` php
#[PostMapping]
#[ResponseBody]
#[MapEntity(User::class)]
public function create(User $user): array {
    // $user is automatically populated from request body
    $this->userRepo->save($user);
    return ['user' => $user->toArray()];
}
```

The mapper matches request parameters to entity setters:
- Parameter `name` → `setName()` method
- Parameter `email` → `setEmail()` method
- Parameter `user_age` → `setUserAge()` method

## Putting It All Together

Here's a complete example of a Product API with MVC architecture:

**Entity (App/Domain/Product.php):**

``` php
namespace App\Domain;

class Product {
    public function __construct(
        public ?int $id = null,
        public string $name = '',
        public string $category = '',
        public float $price = 0,
        public int $stock = 0
    ) {
    }

    public function toArray(): array {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'category' => $this->category,
            'price' => $this->price,
            'stock' => $this->stock
        ];
    }
}
```

**Repository (App/Infrastructure/Repository/ProductRepository.php):**

``` php
namespace App\Infrastructure\Repository;

use App\Domain\Product;
use WebFiori\Database\Repository\AbstractRepository;

class ProductRepository extends AbstractRepository {
    protected function getTableName(): string {
        return 'products';
    }

    protected function getIdField(): string {
        return 'id';
    }

    protected function toArray(object $entity): array {
        return [
            'id' => $entity->id,
            'name' => $entity->name,
            'category' => $entity->category,
            'price' => $entity->price,
            'stock' => $entity->stock
        ];
    }

    protected function toEntity(array $row): Product {
        return new Product(
            (int) $row['id'],
            $row['name'],
            $row['category'],
            (float) $row['price'],
            (int) $row['stock']
        );
    }

    public function findByCategory(string $category): array {
        $result = $this->getDatabase()->table($this->getTableName())
            ->select()
            ->where('category', $category)
            ->execute();

        return array_map(fn($row) => $this->toEntity($row), $result->fetchAll());
    }

    public function findLowStock(int $threshold = 10): array {
        $result = $this->getDatabase()->table($this->getTableName())
            ->select()
            ->where('stock', $threshold, '<')
            ->execute();

        return array_map(fn($row) => $this->toEntity($row), $result->fetchAll());
    }
}
```

**API Controller (App/Apis/ProductController.php):**

``` php
namespace App\Apis;

use App\Domain\Product;
use App\Infrastructure\Repository\ProductRepository;
use WebFiori\Framework\App;
use WebFiori\Http\Annotations\AllowAnonymous;
use WebFiori\Http\Annotations\DeleteMapping;
use WebFiori\Http\Annotations\GetMapping;
use WebFiori\Http\Annotations\MapEntity;
use WebFiori\Http\Annotations\PostMapping;
use WebFiori\Http\Annotations\PutMapping;
use WebFiori\Http\Annotations\RequestParam;
use WebFiori\Http\Annotations\ResponseBody;
use WebFiori\Http\Annotations\RestController;
use WebFiori\Http\WebService;

#[RestController('products', 'Product management API')]
class ProductController extends WebService {
    private ProductRepository $repo;

    public function __construct() {
        parent::__construct();
        $this->repo = new ProductRepository(App::getDatabase());
    }

    #[GetMapping]
    #[ResponseBody]
    #[AllowAnonymous]
    #[RequestParam(name: 'category', type: 'string', optional: true)]
    public function list(): array {
        $category = $this->getParamVal('category');
        
        $products = $category 
            ? $this->repo->findByCategory($category)
            : $this->repo->findAll();

        return [
            'products' => array_map(fn($p) => $p->toArray(), $products)
        ];
    }

    #[GetMapping]
    #[ResponseBody]
    #[AllowAnonymous]
    #[RequestParam(name: 'id', type: 'int')]
    public function get(): array {
        $product = $this->repo->findById($this->getParamVal('id'));

        if (!$product) {
            $this->sendResponse('Product not found', self::E, 404);
            return [];
        }

        return ['product' => $product->toArray()];
    }

    #[PostMapping]
    #[ResponseBody]
    #[MapEntity(Product::class)]
    public function create(Product $product): array {
        $this->repo->save($product);
        return [
            'message' => 'Product created',
            'product' => $product->toArray()
        ];
    }

    #[DeleteMapping]
    #[ResponseBody]
    #[RequestParam(name: 'id', type: 'int')]
    public function delete(): array {
        $this->repo->deleteById($this->getParamVal('id'));
        return ['message' => 'Product deleted'];
    }
}
```

## Folder Structure

Recommended project structure for MVC:

```
App/
├── Apis/                          # API Controllers
│   ├── UserController.php
│   └── ProductController.php
├── Domain/                        # Entities
│   ├── User.php
│   └── Product.php
├── Infrastructure/
│   ├── Repository/                # Repositories
│   │   ├── UserRepository.php
│   │   └── ProductRepository.php
│   └── Schema/                    # Table definitions
│       ├── UserTable.php
│       └── ProductTable.php
└── Database/
    ├── Migrations/                # Database migrations
    └── Seeders/                   # Data seeders
```

## Best Practices

1. **Keep entities simple**: Entities should be plain PHP objects without framework dependencies.

2. **Use repositories for all database access**: Never access the database directly from controllers.

3. **One repository per entity**: Each entity should have its own repository.

4. **Use dependency injection**: Pass repositories to controllers rather than creating them inside.

5. **Validate in controllers**: Use request parameters for input validation before passing to repositories.

6. **Return arrays from controllers**: Let the framework handle JSON serialization.

7. **Use transactions for multiple operations**:
   ``` php
   $this->getDatabase()->transaction(function($db) use ($user, $profile) {
       $this->userRepo->save($user);
       $this->profileRepo->save($profile);
   });
   ```

8. **Handle errors gracefully**:
   ``` php
   try {
       $this->repo->save($entity);
       return ['message' => 'Success'];
   } catch (Exception $e) {
       $this->sendResponse('Operation failed', self::E, 500);
       return [];
   }
   ```

## Related Topics

* [Database](learn/database) - Database abstraction layer
* [Web Services](learn/web-services) - Building APIs
* [Migrations and Seeders](learn/migrations) - Schema management
* [Routing](learn/routing) - URL routing
